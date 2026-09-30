---
tier: epic
title: "Task freshness: a rolling review lease for Ready tasks"
goal: "Every visible, non-recurring Ready task can carry a human-confirmed [fresh::
  YYYY-MM-DD]. A task that was never confirmed (every new capture) or was confirmed
  longer ago than its refresh interval is due for review. The interval is 7 days by
  default and can be overridden per task, per note, and in config. Bryan works through
  the due tasks each morning in their source notes with ]s / [s and Alt+F / Alt+Shift+F,
  and the status bar always shows how many tasks are due and how many he refreshed
  today. Every supported keymap and bob capture edit stamps the tasks it rewrites; task
  creation and automation never do.

  "
phases:
  - id: fresh-core
    title: Freshness contract, placement helper, evaluator, and config in bob-cli
    depends_on: []
    size: medium
    description:
      "fresh-core: docs/freshness.md (definition, placement and state rules, conformance
      vectors), the Rust placement helper stamp_fresh / set_refresh, the pure evaluator,
      and the freshness: config block, with a test per vector and parse-invariance tests
      for both Rust task parsers."
  - id: fresh-cli
    title: bob freshness list and seed
    depends_on:
      - fresh-core
    size: medium
    description:
      "fresh-cli: headless review queue (bob freshness list, human and JSON) and a
      guarded, idempotent, staggered cutover seed (bob freshness seed) that aborts on
      any parse change, plus help, README, and CLI tests."
  - id: capture-stamps
    title: bob capture stamps the existing tasks it rewrites
    depends_on:
      - fresh-core
    size: medium
    description:
      "capture-stamps: plan_task_link and the =x in-progress close stamp fresh in the
      same write; creation never stamps; hooks preserve the field; docs and tests
      updated."
  - id: seed
    title: Seed the live vault and mute the fields
    depends_on:
      - fresh-cli
    size: small
    description:
      "seed: install bob from master, dry-run then apply bob freshness seed to ~/bob,
      verify that no task parses differently before and after, add the CSS rule that
      mutes fresh and refresh, and vault-sync."
  - id: ledger-freshness
    title: bob-ledger-tools api v3 freshness namespace and status bar
    depends_on:
      - fresh-core
      - seed
    size: medium
    description:
      "ledger-freshness: the JavaScript evaluator and placement helper on the shared
      vectors, api v3 (api.freshness with stampLine, state, isDue, tier, rank, queue,
      counts), the freshness: config, and a status bar counter that clicks through to
      the next due task."
  - id: nav-review
    title: Review keys in Bob Navigation Hotkeys
    depends_on:
      - ledger-freshness
    size: medium
    description:
      "nav-review: vault-wide next/previous due-task jumps (Ctrl+Alt+J/K, and the
      commands behind ]s / [s), Alt+F refresh with no other change, and Alt+Shift+F
      refresh-and-advance, including counted and Task Link modes."
  - id: nav-stamps
    title: Bob Navigation Hotkeys gestures stamp freshness; Ctrl+Shift+P refresh row
    depends_on:
      - nav-review
    size: medium
    description:
      "nav-stamps: Alt+N, the Ctrl+Shift+P property/lane rows, Ctrl+Shift+M moves, and
      the dependency toggle stamp every open task they rewrite, and a new pinned refresh
      row sets or clears [refresh:: N]."
  - id: cycler-link-stamps
    title: Status cycling and Task Link gestures stamp freshness
    depends_on:
      - ledger-freshness
    size: medium
    description:
      "cycler-link-stamps: task-status-cycler (Alt+[ / Alt+] to an open status, leaving
      Blocked, reopening) and block-id-prompt (Ctrl+Shift+Enter and ^^ when they rewrite
      the task line) stamp through api.freshness.stampLine."
  - id: vault-review
    title: Review note, dash chip, vim maps, chores, and config
    depends_on:
      - nav-review
      - seed
    size: small
    description:
      "vault-review: freshness.md review note, a REVIEW dash chip, ]s / [s vimrc maps,
      the Morning review and Weekly prune chore text, and the freshness: block in the
      chezmoi-managed config.yml."
  - id: rollout
    title: Install, end-to-end check, glossary term, and Bryan's checklist
    depends_on:
      - capture-stamps
      - nav-stamps
      - cycler-link-stamps
      - vault-review
    size: small
    description:
      "rollout: install bob and confirm plugin deploys, run the headless end-to-end
      checks, add the Task Freshness glossary strand, finalize the docs Surfaces table,
      and hand Bryan his visual checklist and tuning steps."
proposed_by: bbugyi200.apollo.research.v.linker.w0
create_time: 2026-09-30 19:32:04
status: wip
---

- **PROMPT:**
  [prompts/202609/task_freshness_review.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/task_freshness_review.md)

# Plan: Task freshness — a rolling review lease for Ready tasks

## Context

Bryan used to read the whole READY section of `~/bob/dash.md` every morning, trying to
get each project down to five or fewer Ready tasks. That habit guaranteed he saw every
new capture. For example, "pick up our daughter" typed into Google Keep during a walk
and pulled with `bob gkeep pull` the next morning would get seen. But it also re-read
about 180 unchanged tasks a day. Epic `bob-cli-2y` (sticky Next/Pending lanes, Today
read from the ledger; `plan:202609/retire_now_sticky_lanes.md`) turned the morning
review into PENDING → NEXT and left this requirement unmet.

This epic implements **task freshness**. Each Ready task records the last date a human
confirmed that it still needs doing and still looks right. Anything never confirmed, or
confirmed too long ago, is due for review. The morning review then costs roughly "pool ÷
interval + arrivals" glances instead of the whole pool.

The design follows the research report
`research:202609/ready_task_freshness_review/ready_task_freshness_review.md` ("the
report"). Every phase reads it first with `sase artifact read` (run
`sase repo open research -r "<why>"` first if the reference does not resolve). This plan
adopts the report's adjusted requirements R1–R15 and records every place it departs from
them. Where this plan and the report disagree, this plan wins.

### Verified facts this design rests on (see the report's V1–V12)

- **Parsers stop at unknown fields.** Obsidian Tasks 8.4.0 and both bob-cli parsers read
  fields from the end of the line and stop at the first key they don't recognize. The
  two Rust parsers are `parse_details` / `try_take_dataview_field` in
  `src/native/dataview/tasks/task.rs` and `task_metadata` in
  `src/native/task_status_hooks/parse.rs`. A `[fresh::]` appended at the end would
  therefore hide `created`, `priority`, `scheduled`, `id` and `dependsOn`, and Blocked
  would no longer be derived. Every existing Bob inserter appends new fields at the end
  or just before `^id`: `upsertBulletProperty` in bob-navigation-hotkeys and
  `task_metadata_insertion_offset` in `src/native/projects/edits.rs`. So `fresh` needs
  its own placement helper, one per language.
- **Tasks live in their notes.** They sit directly in project and area notes; no task
  line carries a `parent` or `project` field. So a note-level override applies by
  residence.
- **`created` is not the arrival date.** `bob gkeep pull` sets `created` from the Keep
  note's creation time (`src/native/gkeep/render.rs`), so a missing stamp, not
  `created`, has to be the "new" signal.
- **Supporting facts:**
  - The vault uses Tasks' Dataview format (`taskFormat: dataview`), with the global
    filter `#task`.
  - `note_tasks::clean_description` already strips `[k:: v]` fields from capture pickers
    and `bob plan` text.
  - `bob task-status-hooks` only swaps the checkbox byte, so it preserves extra fields.
- **The pool today:**
  - 185 visible open `[ ]` tasks, 10 of them recurring;
  - about 290 `[?]`, 25 `[*]` and 63 `[/]`;
  - `gkeep_inbox.md` holds 65 Ready tasks and `sase.md` 49.
- **Nothing to rename.** No `fresh`, `refresh` or `task_refresh` exists anywhere yet.
  The keys Alt+F, Alt+Shift+F, Ctrl+Alt+J/K, `]s` and `[s` are all unbound. `review.md`
  is taken (a legacy Zorg note); `freshness.md` is free.

### Repositories and surfaces

| Surface          | Where                                            | How to open                                                                                                 |
| ---------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `bob` CLI        | this repo (`bob-cli`)                            | your workspace                                                                                              |
| Obsidian plugins | `bob-plugins`                                    | `sase repo open bob-plugins -r "<why>"`; read its `AGENTS.md`                                               |
| Vault            | `~/bob` (git-synced by `bob-vault-sync.service`) | edit in place; never commit the vault by hand; finish with `bob vault-sync run` and `bob vault-sync status` |
| Config           | chezmoi source for `~/.config/bob/config.yml`    | `sase repo open chezmoi -r "<why>"`                                                                         |
| Research report  | `research` sidecar                               | `sase artifact read research:202609/ready_task_freshness_review/ready_task_freshness_review.md "<why>"`     |

Bob Mac Capture needs no change: bob-cli does the stamping, as the accepted decision
`mac-capture-is-a-thin-client` requires.

Epic `bob-cli-2z` (Work Log entries on the `=x` close) is still landing in
`src/native/capture_pomodoro_close/`. Rebase `capture-stamps` onto it; do not undo it.

## Design

### Vocabulary

- **Task freshness** (aka **freshness**) is the local calendar date a human last
  confirmed that an open task still needs doing as written: its wording, priority,
  project, schedule and dependencies. It is stored as `[fresh:: YYYY-MM-DD]`.
- **Stamp / refresh:** write today's date into `fresh`.
- **Due for review:** an in-scope task in state NEW, RESURFACED or STALE (defined
  below).
- **Refreshed today:** a task whose `fresh` equals today.

### Fields, overrides, config

| What                           | Where                                          | Value                                                       |
| ------------------------------ | ---------------------------------------------- | ----------------------------------------------------------- |
| `[fresh:: YYYY-MM-DD]`         | task line, before the trailing Tasks suffix    | optional; absence means "never confirmed"                   |
| `[refresh:: N]`                | task line, immediately after `fresh`           | optional integer days, 1–365                                |
| `task_refresh: N`              | frontmatter of the note that contains the task | optional integer days, 1–365, for every task in it          |
| `freshness.interval`           | `~/.config/bob/config.yml`                     | integer days, 1–365, default 7                              |
| `freshness.stale_daily_budget` | `~/.config/bob/config.yml`                     | optional integer ≥ 1, default off (a meter, never a filter) |

- **Interval precedence:** `interval(t)` is the task's `refresh`, then the containing
  note's `task_refresh`, then `freshness.interval`, then 7.
- **Invalid values:**
  - An invalid task or note value is linted and falls through to the next level.
  - An invalid `freshness:` block is a config error. `bob freshness` exits 2;
    ledger-tools falls back to the defaults and marks them `invalid`.
  - Unknown keys are ignored, and a mistyped `freshness:` must not break any other
    config loader (mirror how `plan:` is loaded as a raw `serde_yaml::Value`).

### Placement rule (one helper per language, pinned by shared vectors)

The Rust helper is `stamp_fresh` / `set_refresh`; the JavaScript one is
`api.freshness.stampLine` / `setRefreshLine` in bob-ledger-tools.

1. **Scope.** The helpers handle Tasks' Dataview format only, which is the vault's
   format. `bob freshness` refuses to run when the vault's Tasks settings use another
   format.
2. **The Tasks suffix.** Scan from the end of the line, the way both parsers do:
   - first an optional trailing ` ^id` (the `BLOCK_LINK` grammar);
   - then repeatedly whitespace plus one of:
     - a `[k:: v]` or `(k:: v)` field whose key is a Tasks key
       (`priority start created scheduled due completion cancelled repeat onCompletion id dependsOn`),
       whatever its value, following `trailing_inline_field`'s grammar;
     - a `fresh` or `refresh` field;
     - a trailing tag (the `HASH_TAG_AT_END` grammar).
   - Never scan past the start of the task body. If the body begins with the global
     filter token `#task`, never scan past the end of that token.

   The **Tasks suffix** starts at the leftmost Tasks element of that run (a Tasks-key
   field, a tag, or `^id`). `fresh` / `refresh` fields at the run's left edge are not
   part of it.

3. **Canonical output.**
   - Remove every `fresh` and `refresh` field on the line. Each removal also collapses
     the whitespace it leaves to a single space.
   - Rebuild as
     `head.trim_end() + " " + "[fresh:: D]" + (" [refresh:: N]" if a refresh value is kept) + (" " + suffix if there is a suffix)`.
   - The kept refresh value is the first valid existing one; `set_refresh` replaces or
     removes it.
   - The suffix bytes themselves are never changed.
4. **No churn.** If the canonical output equals the input byte-for-byte (the line is
   already stamped today and canonical), report `changed: false`.
5. **Refusals.** Return the line unchanged with a reason for:
   - a line that is not a task line;
   - a recurring task (a `repeat` field in the suffix): its recurrence resurfaces it,
     and Tasks would copy the stamp into the next occurrence;
   - done and cancelled tasks, which callers must not stamp.
6. **Invariance.** For every vector, the Tasks fields both Rust parsers extract are
   identical before and after: status, dates, priority, recurrence, `id`, `dependsOn`,
   tags, block ID. So are the Obsidian Tasks fields the JavaScript tests can check.

### Evaluation (computed at read time, never stored)

```text
today         = the vault's local calendar date (BOB_NOW in tests)
in_scope(t)   = status type TODO ("[ ]") ∧ lane-visible ∧ ¬recurring
                ∧ ¬in a canonical daily note (YYYY/YYYYMMDD.md) ∧ ¬Today(t)
                  lane-visible = the NEXT/PENDING lane predicate on each side: not done,
                  not dependency-blocked, no #hide, not under _templates or _conflicts,
                  no scheduled date after today
fresh(t)      = the latest valid `fresh` date on the line; none if there is none
                (malformed ⇒ ignored + lint; a future date ⇒ treated as none + lint)
state(t)      = NEW         if no fresh(t)
              | RESURFACED  if scheduled(t) exists ∧ fresh(t) < scheduled(t) ≤ today
              | STALE       if today ≥ fresh(t) + interval(t)   (stamped Mon at 7 ⇒ due next Mon)
              | FRESH       otherwise
due_on(t)     = RESURFACED: scheduled(t); STALE/FRESH: fresh(t) + interval(t); NEW: none
due(t)        = in_scope(t) ∧ state(t) ≠ FRESH
tier(t)       = NEW | DUE (RESURFACED and STALE together)
queue order   = NEW by (path, line); then DUE by (due_on, path, line)
counts        = due, new, resurfaced, stale, fresh (in-scope FRESH),
                refreshed_today (tasks of any status, outside _templates/_conflicts,
                whose fresh(t) == today), budget, budget_met
                (budget set ∧ refreshed_today ≥ budget ∧ new == 0)
```

- **The tickler.** RESURFACED makes a short deferral (for example a P1 roll of 2–7 days)
  due as soon as it returns, without any hooks write.
- **Line numbers** are 1-based in JSON and docs. Tasks' `lineNumber` is 0-based, so
  convert it.
- **Lints:**
  - `fresh_malformed`, `fresh_future`, `fresh_duplicate`, `fresh_misplaced` (a
    `fresh`/`refresh` inside the Tasks suffix; the next stamp repairs it);
  - `refresh_invalid` (per task) and `task_refresh_invalid` (per note);
  - `today_link_unresolved` passes through unchanged from the Today engine.

### Who stamps

**Rule:** a supported human gesture that already rewrites an open task's line stamps
that task in the same write, as the last transformation of the line, if the task is
still open and non-recurring afterwards. Stamps apply in every lane, because a Next,
Pending or Blocked task may come back to Ready later. A stamp dated today on a canonical
line is a no-op.

| Surface                | Stamps                                                                                                                                                                                                                                                                                                                 | Never stamps                                                                                                                            |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| bob-navigation-hotkeys | Alt+F and Alt+Shift+F (their only change); Alt+N commit and release; the Ctrl+Shift+P priority, scheduled, dependsOn, delete-property (Ctrl+D), lane and new refresh rows in single, counted and Task Link mode; each open task moved by Ctrl+Shift+M; the `!` dependency toggle when it rewrites the parent task line | the cancel row, project-frontmatter edits, create-project-note-from-task                                                                |
| task-status-cycler     | Alt+[ / Alt+] (including counted and transcluded targets) when the result is an open status, including leaving Blocked by hand; Ctrl+Enter reopening a done task                                                                                                                                                       | closing (done or cancelled), Ctrl+Shift+] bullet → `#task` (that is creation), the dependency-ID normalizer, `recoverBlockedDependents` |
| block-id-prompt        | Ctrl+Shift+Enter and `^^` when they rewrite the task line (Ready/Blocked → Next, a new block ID)                                                                                                                                                                                                                       | unlink, Task Link removal, Ctrl+6 rename                                                                                                |
| `bob capture`          | `plan_task_link` (the link direction of `@route+id!`, Ensure Next, solo `@route:id` / `^route:id`, link-then-close) and the `=x` rows that set `[/]`                                                                                                                                                                   | new tasks on any route, `=x` complete, unlink, start rows, sub-bullets                                                                  |
| Automation             | —                                                                                                                                                                                                                                                                                                                      | hooks, `projects sync`, `randomize`, `gkeep pull`, `highlights`, `move-done-tasks`, `nightly`, `vault-sync`, `capture-task-id`          |
| `bob freshness seed`   | the one-time cutover (a documented exception)                                                                                                                                                                                                                                                                          | —                                                                                                                                       |
| Hand editing           | —                                                                                                                                                                                                                                                                                                                      | not monitored; edit, then press Alt+F                                                                                                   |

- **Deferrals.** Scheduling a task into the future (a P-level roll or a date) stamps it,
  and it becomes Blocked. When it returns it is RESURFACED, so Bryan sees it again.
- **Automatic returns.** A Blocked → Ready return through the hooks (the scheduled date
  arrives, or a dependency closes) never stamps.
- **One JavaScript helper.** Plugins normally copy small pure helpers instead of
  importing each other. Placement is the exception: nav, task-status-cycler and
  block-id-prompt call `api?.freshness?.stampLine?.(line, dateText) ?? line` on
  bob-ledger-tools (api `version >= 3`).
  - This is safe because the two failure modes are not equal. A missing stamp only means
    Bryan sees the task once more; a misplaced stamp hides Tasks fields.
  - So the risky part lives in one place. When ledger-tools is absent or old, the
    gesture simply doesn't stamp.
  - Pure planners take the stamper as an injected option (identity by default), so they
    stay testable. Tests may `require` ledger-tools' real helper.
  - Say this in a source comment at each call site and in the ledger-tools README api
    section.

### The review experience

- **Status bar** (bob-ledger-tools, desktop): `⟳ 23 due · 3 new · ✓ 12 today`.
  - With a budget it reads `✓ 12/15`.
  - Visual states:
    - an accent while `new > 0`;
    - a muted "clear" style when nothing is due;
    - a budget-met style when `budget_met` holds.
  - The tooltip gives the NEW / RESURFACED / STALE breakdown and the most days overdue.
  - Clicking runs `bob-navigation-hotkeys:jump-to-next-due-task`, falling back to
    opening `freshness.md`.
- **Jumps** (bob-navigation-hotkeys):
  - The commands are `jump-to-next-due-task` / `jump-to-prev-due-task`. Their default
    hotkeys are Ctrl+Alt+J / Ctrl+Alt+K, and the vimrc maps them to `]s` / `[s` ("s" for
    stale).
  - They are vault-wide and follow `api.freshness.queue()`. They open the source note,
    put the cursor on the task and center it.
- **Refresh keys:**
  - **Alt+F** stamps the cursor task, or a Task Link's target, plus the next N tasks
    when counted, and changes nothing else.
  - **Alt+Shift+F** stamps, then jumps to the next due task. It is the one-key-per-task
    path through the review.
- **Review note** `~/bob/freshness.md`: a counts line and one Tasks block grouped NEW →
  DUE. Headless `bob query` sees that block as empty, just as it sees TODAY;
  `bob freshness list` is the headless view.
- **Dash:** one chip, `REVIEW 23 · 3 new · ✓ 12`, which links to `freshness.md` and is
  highlighted while `new > 0`. The TODAY / PENDING / NEXT / READY sections stay mutually
  exclusive and unchanged, as the accepted decision `today-is-read-from-the-ledger`
  requires.
- **Review outcomes**, one key each (every row except "edit" stamps by itself):

  | Decision                    | Key                                                 |
  | --------------------------- | --------------------------------------------------- |
  | still right, keep           | Alt+Shift+F (or Alt+F)                              |
  | keep, but see it less often | Ctrl+Shift+P → refresh → 14 / 30 / 90               |
  | not now                     | Ctrl+Shift+P priority (rolls a P-level `scheduled`) |
  | do today / commit           | Ctrl+Shift+Enter / Alt+N                            |
  | route to a project          | Ctrl+Shift+M                                        |
  | drop                        | Ctrl+Shift+P cancel                                 |
  | wording is wrong            | edit, then Alt+F                                    |

- **Morning ritual** (it replaces reading READY):
  1. Run `bob gkeep pull` (the existing chore).
  2. Walk `]s` / Alt+Shift+F until the status bar shows **0 new**. This step is never
     capped and never skipped.
  3. Continue through DUE until 0 due, or until the budget meter is met.
  4. Then PENDING → NEXT: link today's work and release the rest. READY becomes a pull
     list, not reading material.
- **Weekly ritual:** one more line in the existing Weekly prune chore. Clear any
  leftover DUE, or lengthen that note's `task_refresh`, and look for projects with no
  Next or Ready task.

### Contracts

- **`bob freshness list -f json`** (`schema_version: 1`):

  ```json
  {
    "ok": true,
    "schema_version": 1,
    "date": "2026-10-08",
    "config": { "interval": 7, "stale_daily_budget": null },
    "counts": {
      "due": 23,
      "new": 3,
      "resurfaced": 2,
      "stale": 18,
      "fresh": 150,
      "refreshed_today": 12,
      "budget": null,
      "budget_met": false
    },
    "queue": [
      {
        "rank": 1,
        "tier": "new",
        "state": "new",
        "path": "gkeep_inbox.md",
        "line": 14,
        "block_id": null,
        "status_symbol": " ",
        "text": "Pick up our daughter",
        "created": "2026-09-30",
        "fresh": null,
        "interval": 7,
        "interval_source": "config",
        "due_on": null,
        "days_overdue": null
      }
    ],
    "warnings": [
      {
        "code": "fresh_misplaced",
        "path": "sase.md",
        "line": 88,
        "message": "…"
      }
    ]
  }
  ```

  `interval_source` is one of `task`, `note`, `config` or `default`, and `text` is
  `clean_description`.

- **`bob freshness seed -f json`:**
  - `{ok, schema_version: 1, date, dry_run, stamped: {ready, other}, buckets: [{fresh, due_on, count, notes}], skipped: {already_stamped, recurring, out_of_scope}, files: [...], warnings}`.
- **bob-ledger-tools api `version: 3`.** It is additive: every v2 member is unchanged,
  and v2 callers such as nav's `readLaneBudgets` (`>= 2`) keep working. It adds:

  ```text
  api.freshness = Object.freeze({
    version: 1,
    config()                         -> {interval, staleDailyBudget, invalid}
    stampLine(line, dateText?)       -> string   // canonical placement; identity when refused
    setRefreshLine(line, days|null, dateText?) -> string // also stamps
    state(task)                      -> "new" | "resurfaced" | "stale" | "fresh" | null (out of scope)
    isDue(task) -> boolean;  tier(task) -> "1 · NEW" | "2 · DUE" | ""
    rank(task)                       -> queue index, or Number.MAX_SAFE_INTEGER
    intervalFor(task)                -> {days, source}
    queue()                          -> [{key, path, line, lineNumber, text, originalMarkdown, blockId, state, tier, fresh, dueOn, daysOverdue, interval}]
    counts()                         -> {due, new, resurfaced, stale, fresh, refreshedToday, budget, budgetMet}
    lints()                          -> [{code, path, line, message}]
  })
  ```

  Every member is synchronous, never awaits and never throws.

- **Capture JSON stays `schema_version: 1`.** The rewritten `task_line` may now contain
  `[fresh:: …]`; `text` fields stay clean.

## Cross-cutting rules for every phase

- **Read first:** the report (`sase artifact read`), this plan's Design section, and, in
  bob-cli, `docs/freshness.md` once `fresh-core` has landed.
- **New or changed CLI surface:** read `sase/memory/cli_rules.md` with
  `/sase_memory_read`. That means alphabetical subcommands and options, a short alias
  for every public long option, excellent `--help`, and color on a TTY only.
- **bob-cli:**
  - `just all` (fmt, lint, test) must pass.
  - Keep new Rust files under about 1500 lines.
  - Update `README.md`, `docs/README.md` and `docs/*.md` in the phase that changes
    behavior. Each phase fills only its own row of the `docs/freshness.md` Surfaces
    table.
- **bob-plugins:**
  - `npm test` and `npm run validate` must pass.
  - Register any new test file in `package.json`.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`; the
    default repo path may not exist on this host.
- **Vault:** edit in place, keep each file's formatting, never touch daily notes or
  `done/`, and finish with `bob vault-sync run` plus `bob vault-sync status`.
- **Memory:** only the glossary strand in `rollout`, through `/sase_memory_write` (Edit
  And Republish) and then `sase memory init`. This plan's approval authorizes that edit.
- **Epic workers** record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.

## Phase: fresh-core — Freshness contract, placement helper, evaluator, and config in bob-cli

**Docs (`docs/freshness.md`, new).** Title it "Task freshness". This file is the
contract that both implementations cite. Sections:

1. **Definition:** the Vocabulary above, and why a missing stamp, not `created`, means
   new (gkeep's `created`).
2. **Fields, overrides and config:** the table above, with the precedence rule and an
   example `freshness:` block.
3. **Placement rule:** rules 1–6 above, verbatim in substance, including the Tasks key
   list and the global-filter floor.
4. **Evaluation:** scope, state, `due_on`, tiers, queue order, counts and lints.
5. **Who stamps:** the table above.
6. **Review ritual:** morning and weekly, as above.
7. **`bob freshness`:** a placeholder that `fresh-cli` fills.
8. **Surfaces:** one row per surface (`bob freshness`, `bob capture`, bob-ledger-tools,
   bob-navigation-hotkeys, task-status-cycler, block-id-prompt, `freshness.md`,
   `dash.md`), each marked with the phase that fills it.
9. **Placement conformance examples** and **State conformance examples** (below). The
   bob-ledger-tools JavaScript tests use them verbatim; say so.

Add `docs/freshness.md` to the `docs/README.md` index. Add a short "Task freshness"
paragraph to `README.md` that links to it; the command-table row waits for `fresh-cli`.

**Placement conformance examples** (D = `2026-10-08`). Each gives the input line, the
expected line, and `changed`:

- **P1 bare:** `- [ ] #task Buy milk` → `- [ ] #task Buy milk [fresh:: 2026-10-08]`
- **P2 created:** `- [ ] #task Buy milk [created::2026-09-29]` →
  `- [ ] #task Buy milk [fresh:: 2026-10-08] [created::2026-09-29]`
- **P3 block ID only:** `- [ ] #task Pick up Abby ^pickup` →
  `- [ ] #task Pick up Abby [fresh:: 2026-10-08] ^pickup`
- **P4 interleaved suffix:**
  `- [ ] #task Plan trip [created::2026-08-26] #hide [priority:: high] ^trip` →
  `- [ ] #task Plan trip [fresh:: 2026-10-08] [created::2026-08-26] #hide [priority:: high] ^trip`
- **P5 unknown field stays left:**
  `- [ ] #task Read X [[#^h-8bac|🔖]] [h:: e629] [created::2026-08-28]` →
  `- [ ] #task Read X [[#^h-8bac|🔖]] [h:: e629] [fresh:: 2026-10-08] [created::2026-08-28]`
- **P6 spacing:**
  `- [ ] #task Rahway  [created:: 2026-07-15]  [scheduled:: 2026-08-10] ^rahway` →
  `- [ ] #task Rahway [fresh:: 2026-10-08] [created:: 2026-07-15]  [scheduled:: 2026-08-10] ^rahway`
  (the head collapses to one space; the suffix bytes are untouched).
- **P7 restamp:** `- [ ] #task Buy milk [fresh:: 2026-10-01] [created::2026-09-29]` →
  the same line with `[fresh:: 2026-10-08]`.
- **P8 same day:** `- [ ] #task Buy milk [fresh:: 2026-10-08] [created::2026-09-29]` →
  byte-identical, `changed: false`.
- **P9 misplaced:** `- [ ] #task Buy milk [created::2026-09-29] [fresh:: 2026-10-01]` →
  `- [ ] #task Buy milk [fresh:: 2026-10-08] [created::2026-09-29]`. The reader reports
  `fresh_misplaced` on the input.
- **P10 duplicates:**
  `- [ ] #task A [fresh:: 2026-09-01] B [fresh:: 2026-09-20] [created::2026-09-01]` →
  `- [ ] #task A B [fresh:: 2026-10-08] [created::2026-09-01]`. The reader reports
  `fresh_duplicate` and uses 2026-09-20.
- **P11 refresh follows fresh:**
  `- [ ] #task Rename queue input [refresh:: 14] [created::2026-09-10] [priority:: low]`
  →
  `- [ ] #task Rename queue input [fresh:: 2026-10-08] [refresh:: 14] [created::2026-09-10] [priority:: low]`
- **P12 set / clear refresh:** `set_refresh(P2 output, 30)` →
  `- [ ] #task Buy milk [fresh:: 2026-10-08] [refresh:: 30] [created::2026-09-29]`, and
  `set_refresh(…, none)` removes it.
- **P13 recurring refused:**
  `- [ ] #task Water plants [repeat:: every week] [created::2026-09-01]` → unchanged,
  refused as `recurring`.
- **P14 global-filter floor:** `- [ ] #task #hide ^x` →
  `- [ ] #task [fresh:: 2026-10-08] #hide ^x`, and `- [ ] #task #prj Ship it #hide ^prj`
  → `- [ ] #task #prj Ship it [fresh:: 2026-10-08] #hide ^prj`
- **P15 parenthesized field:** `- [ ] #task Call mom (created:: 2026-09-01)` →
  `- [ ] #task Call mom [fresh:: 2026-10-08] (created:: 2026-09-01)`
- **P16 indented Blocked:**
  `\t- [?] #task Deferred [created::2026-09-01] [scheduled:: 2026-10-20]` →
  `\t- [?] #task Deferred [fresh:: 2026-10-08] [created::2026-09-01] [scheduled:: 2026-10-20]`
- **P17 done refused:** `- [x] #task Old [completion:: 2026-10-01]` → unchanged, refused
  as `closed`.

**State conformance examples** (today `2026-10-08`, config interval 7, and a Ready,
visible, non-recurring task in `a.md` unless noted):

- **S1 new:** no `fresh` → `new`.
- **S2 fresh:** `fresh 2026-10-02` → `fresh`, `due_on 2026-10-09`.
- **S3 boundary:** `fresh 2026-10-01` → `stale`, `due_on 2026-10-08`, `days_overdue 0`.
- **S4 overdue:** `fresh 2026-09-20` → `stale`, `due_on 2026-09-27`, `days_overdue 11`.
- **S5 task beats note:** `[refresh:: 14]`, note `task_refresh: 3`, `fresh 2026-10-01` →
  `fresh` (task source, due 2026-10-15).
- **S6 note beats config:** note `task_refresh: 3`, `fresh 2026-10-05` → `stale` (note
  source).
- **S7 config:** `freshness.interval: 10`, `fresh 2026-10-01` → `fresh` (config source,
  due 2026-10-11).
- **S8 invalid overrides fall through:** `[refresh:: 0]` → `refresh_invalid`, and note
  `task_refresh: soon` → `task_refresh_invalid`. Both fall through to config 7.
- **S9 malformed:** `[fresh:: 2026-13-01]` → `new` + `fresh_malformed`.
- **S10 future:** `[fresh:: 2026-10-09]` → `new` + `fresh_future`.
- **S11 resurfaced:** `fresh 2026-10-05`, `scheduled 2026-10-07` → `resurfaced`,
  `due_on 2026-10-07`.
- **S12 not resurfaced:** `fresh 2026-10-07`, `scheduled 2026-10-07` → `fresh`.
- **S13 out of scope** (state `null`):
  - a recurring task;
  - `[?]` with a future `scheduled`;
  - `[*]`; `[/]`;
  - `#hide`;
  - `_templates/x.md`;
  - daily note `2026/20261008.md`;
  - a Ready task linked under today's open Pomodoro;
  - a `[ ]` whose `dependsOn` names an open task.
- **S14 queue order.** Given:
  - NEW `b.md:3` and NEW `a.md:9`;
  - STALE due 2026-10-01 at `c.md:2`;
  - RESURFACED due 2026-10-07 at `a.md:4`;
  - STALE due 2026-10-07 at `a.md:2`.

  The order is `a.md:9`, `b.md:3`, `c.md:2`, `a.md:2`, `a.md:4`.

- **S15 counts:**
  - `refreshed_today` counts a `[*]` and an `[x]` stamped 2026-10-08, but not a Ready
    task stamped 2026-10-07.
  - With budget 15, 15 refreshed and 0 new → `budget_met: true`; with 1 new → `false`.

**Rust code.** Create a new module `src/native/freshness/` and register it in
`src/native.rs`:

- **`placement.rs`:**
  - `stamp_fresh(line, date) -> Stamp { line, changed, refused: Option<Refusal> }` and
    `set_refresh(line, Option<u16>, date) -> Stamp`;
  - `read_freshness(line, today) -> FreshRead { fresh, refresh, lints }`;
  - `tasks_suffix_start(line) -> usize`.
  - Build on `task_fields::inline_fields`; don't write a third field lexer.
- **`state.rs`:**
  - the pure evaluator over a small input row: path, 1-based line, status symbol and
    type, recurring, lane-visible, daily note, Today member, scheduled, created, the raw
    line, and the note's raw `task_refresh`;
  - it returns state, fresh, interval and source, `due_on`, `days_overdue` and lints;
  - also `queue(rows, today, config)` and `counts(rows, today, config)`.
- **Config:** `src/native/config/freshness.rs`, modeled on `config/plan.rs`:
  - `FreshnessConfig { interval: u16 = 7, stale_daily_budget: Option<u32> }` and
    `load_freshness_config(path)`;
  - `freshness` read as `Option<serde_yaml::Value>` on `RawConfig`;
  - the same null / not-a-mapping / invalid-value handling and tests as `plan.rs`,
    including "a mistyped `freshness:` leaves other loaders working".

**Tests.**

- One unit test per P and S vector.
- An invariance test that runs every P vector's input and output through both
  `dataview/tasks/task.rs` (`Task::from_line` / `parse_details`) and
  `task_status_hooks/parse.rs::task_metadata`. Every Tasks field must be equal.
- A hooks test showing that a stamped `[?]` task with `dependsOn` or a future
  `scheduled` is still derived Blocked, and that a checkbox swap preserves
  `[fresh:: …]`.

## Phase: fresh-cli — `bob freshness list` and `seed`

**Rows.**

- Add a `pub(crate)` rich query next to `query_matching_descriptions` in
  `src/native/dataview/tasks/mod.rs`. It returns per task:
  - path and 1-based line;
  - status symbol and type;
  - `original_markdown`;
  - `created` and `scheduled`;
  - `is_recurring`, `is_blocked`, tags and `block_id`.
- Add a `READY_QUERY` built exactly like `NEXT_QUERY` with `status.type is TODO` (apply
  the same whole-token `#hide` predicate the lane counts use). Use a looser open-task
  query for `refreshed_today` and the seed.
- Read each note's `task_refresh` with `projects::scan::parse_frontmatter` /
  `frontmatter_value`, once per note.
- Get Today from `plan_budget::today::today_tasks`; an absent daily note means an empty
  Today.
- Refuse with exit 2 when the vault's Tasks settings are not the Dataview format.

**`bob freshness`** (clap builder, modeled on `src/native/projects/mod.rs`):

- Add it to `SUBCOMMANDS` in `src/runner.rs` in alphabetical order, with an `AFTER_HELP`
  example.
- `list` (the default when no subcommand is given, as in `bob plugins`):
  - options `-f/--format human|json` and `-l/--limit N` (rows only; counts are always
    whole);
  - human output, colored only on a TTY:

    ```text
    bob freshness · Thu 2026-10-08 · every 7d

      REVIEW 23 due · 3 new · 2 resurfaced · 18 stale · ✓ 12 today

      NEW
        gkeep_inbox.md:14   Pick up our daughter          created 2026-09-30
      DUE
        a.md:2              Rename queue input            stale 3d · fresh 2026-09-28 · every 7d (note)
        b.md:40             Week habits                   resurfaced · scheduled 2026-10-07
    ```

    With a budget: `✓ 12/15 today`. Lints go last. Exit codes: 0; 1 for I/O errors; 2
    for invalid config or an unsupported format.

- `seed` (`-d/--dry-run`, `-F/--force`, `-f/--format human|json`):
  - **Ready tasks.**
    - Take in-scope Ready tasks with no valid `fresh` and group them by note.
    - Bin-pack the notes into 7 buckets, largest note first into the least-loaded
      bucket. Split a note bigger than `ceil(total / 7)` into consecutive line-order
      chunks. Break ties by path, then bucket index.
    - Bucket `k` (1..7) gets `today − 7 + k`. Per task, the stamp is
      `max(bucket date, today − interval(t) + 1, scheduled(t) when scheduled(t) ≤ today)`,
      clamped to ≤ today. So nothing is due on cutover day, and nothing is RESURFACED
      right away.
  - **Every other open, non-recurring task** (Next, Pending, Blocked, and out-of-scope
    Ready) outside `#hide`, `_templates`, `_conflicts` and daily notes, with no valid
    `fresh`, gets today. Returning deferrals then arrive as RESURFACED or STALE, never
    as NEW.
  - **Guard against a second seed.**
    - Refuse when any task already has a `fresh` dated before today, unless `-F/--force`
      is given.
    - The message says why: seeding after cutover would mark every capture since then as
      reviewed.
    - A same-day rerun is an idempotent no-op.
  - **Parse-invariance guard.** Before writing anything, re-parse every changed line
    with both Rust parsers. Abort the whole seed with no writes if any Tasks field
    differs, and list the lines.
  - **Writes.**
    - Re-read each file just before writing and refuse if it changed since the scan.
    - Write through a temp file plus rename.
    - On failure, report which files were written; the vault's git history is the
      rollback.
  - **Output:** bucket dates with their `due_on`, counts and per-note counts; the JSON
    contract above.

**Docs and help.**

- Fill `docs/freshness.md` § `bob freshness` and its Surfaces row.
- Add the README command-table row.
- Add the `justfile` `install-smoke` help lines.
- Update the help tests in `tests/cli/help.rs` and `tests/cli/help_options.rs`: the
  alphabetical order, no long-only options, and plain help.

**CLI tests** (new `tests/cli/freshness.rs`). Use a fixture vault with Dataview-format
Tasks settings (`write_blocked_tasks_settings`) and `BOB_NOW`. Cover:

- the S vectors end to end (`note task_refresh`, config, Today exclusion through a daily
  note);
- JSON and human output, with no ANSI when piped;
- `--limit`;
- seed bucketing, the idempotent same-day rerun, the guard and `--force`, `--dry-run`
  writing nothing, the concurrent-change refusal, and the invariance abort (inject a
  line whose output would change a field, or unit-test the guard);
- an emoji-format vault refused with exit 2.

## Phase: capture-stamps — `bob capture` stamps the existing tasks it rewrites

1. **Stamping sites.**
   - `plan_task_link` in `src/native/capture_task_toggle.rs`: when the status or
     scheduled edits changed the line, apply
     `freshness::stamp_fresh(&updated_text, today)` as the last transformation. A line
     the link leaves byte-identical (Next or In Progress with no future `scheduled`) is
     not stamped, so the task note is not written; this matches block-id-prompt. A
     refusal leaves the line as it is. This one site covers:
     - the link direction of `@route+id!`;
     - Ensure Next;
     - solo `@route:id` / `^route:id`;
     - `@route:id=x…` link-then-close.
   - `ClosePlanner::apply_startable` in
     `src/native/capture_pomodoro_close/linked_tasks.rs`: after
     `normalize_task_metadata_spacing`, stamp with the close date.
   - Confirm that no other capture path rewrites an existing task line. Add a stamp to
     any you find that ends open, and list them in the phase notes.
2. **Never on creation.** New-task renderers stay unchanged: `format_task_line` and
   friends, project-note capture, gkeep and highlights. Add a test asserting that a
   routed `@project` capture and a gkeep-rendered task carry no `fresh`.
3. **Reporting.** Existing JSON fields such as `task_line` and `previous_task_line` show
   the stamped line. Add no new fields, and keep `text` clean.
   - Check how Bob Mac Capture renders these fields (`sase repo open bob-mac-capture`,
     or `gh:bobs-org/bob-mac-capture`). If it shows the raw line, record a
     `PROPOSED FOLLOW-UP:` rather than changing the Mac app.
4. **Tests.** Update the capture unit and CLI tests whose expected task lines change.
   Add cases for:
   - a Ready or Blocked → Next link stamps;
   - a Next or In Progress link stamps only when it retires a future `scheduled`, and
     otherwise leaves the task note untouched;
   - an `=x` close to `[/]` stamps;
   - `=x` complete doesn't;
   - a recurring task is never stamped;
   - stamping twice on the same day is byte-identical.
5. **Docs.** Add one sentence each to `docs/capture.md` (in the toggle, Ensure Next and
   `=x` sections) and to the `docs/freshness.md` Surfaces row.

## Phase: seed — Seed the live vault and mute the fields

1. **Install and check the vault.**
   - From an up-to-date bob-cli master checkout, run
     `cargo install --path . --locked --force`.
   - Then run `bob vault-sync run` and `bob vault-sync status`. Stop and record it if
     the vault is conflicted.
2. **Baseline.** Save a before-snapshot of how every open task parses. Use
   `bob query --tasks` JSON over open tasks (status, created, scheduled, priority, `id`,
   `dependsOn`, blocked, tags, path, line), and write it to a temp file.
3. **Dry run.**
   - Run `bob freshness seed --dry-run -f json`, summarize it with `jq`, and paste the
     result into the phase notes: stamped Ready/other counts, the bucket dates with
     their counts, and the biggest notes.
   - Sanity-check against the plan's numbers: roughly 175 Ready and 380 other.
   - If the guard refuses because stamps dated before today exist, stop and record it.
     Do not use `--force` without Bryan.
4. **Apply** `bob freshness seed`, then take the after-snapshot and diff it against the
   baseline. The only allowed differences are descriptions and markdown containing
   `[fresh:: …]`; every parsed field must match. On any other difference, revert the
   seeded files from the vault's git history, run `bob vault-sync run`, and record it.
5. **Check the result:**
   - `bob freshness list` shows `new` equal to only the NEW captures since the seed
     (normally 0);
   - due today is 0;
   - `refreshed_today` equals the other-lane count.
6. **Mute the fields** in `~/bob/.obsidian/snippets/dataview-properties.css`. Mirror the
   existing `data-dv-norm-key` rules (the `dependson` rule's key-span anchoring notes
   apply). Render `fresh` and `refresh` as a small, low-contrast pill: keep them
   readable, don't hide them.
7. **Finish** with `bob vault-sync run` and `bob vault-sync status`. Paste the counts,
   the diff verdict, and the commit into the phase notes.

## Phase: ledger-freshness — bob-ledger-tools api v3 freshness namespace and status bar

This phase works in bob-plugins `plugins/bob-ledger-tools/`: `main.js`, `styles.css`,
and the tests. It waits for `seed` because every plugin stamp flows through this api.
That way no Obsidian gesture can stamp before the cutover seed, which refuses to run
once stamps dated before today exist, and the status bar never opens on a vault where
every task is NEW.

1. **Pure helpers**, exported through `module.exports.helpers`, which mirror
   `docs/freshness.md`:
   - `freshnessStampLine(line, dateText)` and
     `freshnessSetRefreshLine(line, days, dateText)`, returning
     `{line, changed, refused}`;
   - `readFreshness(line, todayText)`, `freshnessIntervalFor(...)`,
     `freshnessState(...)`, `freshnessQueue(rows, ...)`, `freshnessCounts(rows, ...)`;
   - `freshnessBlock(yaml)` and `coerceFreshnessConfig(block)`, beside `planCapsBlock` /
     `coercePlanCaps`, loaded next to `loadPlanCaps` with the same injectable options.
   - Keep the placement key list and grammar in one named constant, with a comment
     citing `docs/freshness.md` and bob-cli's `freshness/placement.rs`.
2. **The evaluator in Obsidian.**
   - Rows come from the Tasks cache (`planBlockTasks(app)`). The `fresh` / `refresh`
     values come from `originalMarkdown`.
   - Frontmatter comes from
     `app.metadataCache.getCache(path)?.frontmatter?.task_refresh`.
   - Visibility is `planLaneVisible` plus status type TODO, `!task.recurrence`, not a
     canonical daily-note path, and `!isTodayTask`.
   - Memoize the evaluated rows and queue on: the identity of the array `getTasks()`
     returns, the local date, a frontmatter generation, and the config. Bump the
     frontmatter generation on metadataCache `changed` when `task_refresh` differs, and
     re-read the config on bumps and at rollover. This keeps `rank(task)` O(1) inside
     Tasks' `sort by function`.
   - **Refresh.** When the due key set changes because of rollover, a frontmatter change
     or a config change (not task edits, which Tasks re-renders itself), trigger
     `TODAY_RELOAD_EVENT` so the review note re-queries.
3. **api v3.**
   - `version: 3` adds `freshness` exactly as in the Contracts section; all v2 members
     stay.
   - `stampLine` / `setRefreshLine` return strings and default `dateText` to today's
     local date.
   - Update the tests that assert `version === 2`, and document v3 plus the placement
     exception in the README api section.
4. **Status bar** (guarded with `typeof this.addStatusBarItem === "function"`; skipped
   on mobile):
   - text, tooltip, states and click as in the Design section;
   - updates debounced (about 150 ms, like `schedulePlanBlockRerender`) on Tasks
     `cache-update`, the frontmatter generation, Today changes, and the 60 s rollover
     tick;
   - shows `⟳ –` while Tasks is unavailable;
   - classes in `styles.css` use theme variables.
5. **Tests** (new `scripts/test-ledger-tools-freshness.cjs`, registered in
   `package.json`):
   - every P and S vector from `docs/freshness.md`, verbatim and cited;
   - config coercion, including an invalid block and unknown keys;
   - memo invalidation on a new `getTasks()` array, rollover and frontmatter change;
   - the reload event firing only when the due key set changes;
   - status bar text in each state, and a missing `addStatusBarItem`;
   - the api v3 shape, and that v2 members are unchanged.
6. **Finish:** README row and api docs, a minor version bump, and a deploy with
   `-p bob-ledger-tools`.

## Phase: nav-review — Review keys in Bob Navigation Hotkeys

This phase works in bob-plugins `plugins/bob-navigation-hotkeys/main.js`.

1. **Commands:**

   | Command id                           | Name                                                  | Default hotkey |
   | ------------------------------------ | ----------------------------------------------------- | -------------- |
   | `jump-to-next-due-task`              | Jump to next task due for freshness review            | Ctrl+Alt+J     |
   | `jump-to-prev-due-task`              | Jump to previous task due for freshness review        | Ctrl+Alt+K     |
   | `refresh-task-freshness`             | Refresh task freshness (confirm it still looks right) | Alt+F          |
   | `refresh-task-freshness-and-advance` | Refresh task freshness and jump to the next due task  | Alt+Shift+F    |

   All of them require ledger-tools api `version >= 3` with a `freshness` namespace.
   Otherwise they show the Notice "Bob Ledger Tools api v3 required".

2. **Jumps.**
   - Read `api.freshness.queue()` fresh on every call.
   - **Position:**
     - from a task that is in the queue, go to the following entry (for next) or the
       preceding one (for previous);
     - from the task Alt+F / Alt+Shift+F just stamped (remember its rank tuple for the
       session, because the Tasks cache lags), go to the entry after that tuple;
     - otherwise go to the first entry (for next) or the last (for previous).
   - Wrap around with a Notice. An empty queue shows
     `Nothing due for review · ✓ 12 today`.
   - **Landing.**
     - Open the note with leaf reuse, put the cursor on the task line, and center it.
       Reuse `focusTaskMoveDestination` / `scheduleOpenTaskJumpCenter`.
     - Resolve the line by checking that the entry's `originalMarkdown` is at `line`,
       else a unique exact match in the file. Otherwise rebuild once and show "Review
       queue changed — try again". Never write.
   - **Notice:** `Review 3/23 · NEW` (or `· stale 4d`, `· resurfaced`).
3. **Alt+F.**
   - Targets are discovered like Alt+N's:
     - on a task line, the task plus the next N tasks (counted `N<Alt+F>`);
     - on a dedicated Task Link, the linked task plus the next N sibling links, resolved
       to their tasks.
   - Refuse non-tasks, done and cancelled tasks, and recurring tasks (Notice
     `recurring · not reviewed`).
   - **Write** `api.freshness.stampLine` and nothing else:
     - single-note targets through the editor transaction;
     - cross-note targets through the Alt+N plan, with the preimage check and rollback.
   - **Notice:** `Fresh ✓ 1 task · 22 due (3 new) · ✓ 13 today`. Take the counts from
     `api.freshness.counts()` before the write and adjust them by the change, because
     the Tasks cache lags. With a budget, use `✓ 13/15`; once the budget is met and new
     is 0, add ` · done for today`.
   - **Vim.** CodeMirror Vim swallows Alt chords. Add capture-phase keydown listeners
     like `registerCountedLaneToggleInputListeners` / `isCountedLaneToggleKeydown`,
     matching `event.code === "KeyF"` with and without Shift, in Vim normal mode, with
     the pending count and the duplicate-dispatch guard.
4. **Alt+Shift+F:** do everything Alt+F does, then jump as `jump-to-next-due-task` from
   the stamped entry's rank tuple, skipping every key just stamped.
5. **Tests** (`scripts/test-navigation-hotkeys.cjs`, or a new
   `scripts/test-navigation-freshness.cjs`, registered). Cover:
   - queue position (in the queue, just stamped, elsewhere, wrap, empty);
   - line resolution, including a moved line and a stale queue;
   - single, counted and Task Link targets;
   - the refusals;
   - Notices with and without a budget;
   - the Alt+F / Alt+Shift+F keydown predicates;
   - a missing api.
6. **Finish:** README row (the four commands and their keys), a minor version bump, and
   a deploy with `-p bob-navigation-hotkeys`.

## Phase: nav-stamps — Bob Navigation Hotkeys gestures stamp freshness; Ctrl+Shift+P refresh row

This phase works in bob-plugins `plugins/bob-navigation-hotkeys/main.js`.

1. **Inject the stamper.** Add a `stampLine` option (identity by default) to the pure
   planners, and pass `api.freshness.stampLine` with today's date at the command layer.
   Apply it as the **last** edit of every rewritten task line that stays open. The
   planners:
   - `planTaskLaneBatch` (Alt+N commit and release);
   - `planCountedBulletPropertyBatch`, the single-task property writers
     (`setInlineBulletPropertyValues`, `setBulletPriorityValue`,
     `deleteBulletPropertyValue`) and the link-picker plans (`commitLinkPickerPlans`);
   - the lane row;
   - `planTaskMoveAcrossFiles` (each moved open task line);
   - the dependency writers (`setLocalTaskDependency`,
     `applyCountedLocalTaskDependency`, `executeDependencyBatch`, and the `!`
     transclusion sync where it rewrites the parent line).
2. **Never stamp** the cancel row, project-frontmatter edits, or
   `createProjectNoteFromTask` / `convertProjectNoteToTask`.
3. **Guard `fresh` and `refresh`.** They must never go through the end-append
   `upsertBulletProperty` / `insertMissingBulletProperty`. Add a guard and a test.
   Because the stamper runs last, a field reordering such as `reorderPropertyNames` can
   never leave `fresh` inside the suffix.
4. **Refresh row** in the Ctrl+Shift+P picker:
   - It is pinned right after the lane row, in task, counted and link mode.
   - Its detail shows the effective interval and source, e.g.
     `refresh · every 7 d (config)` or `every 3 d (note)`.
   - The value stage offers 2, 3, 7, 14, 30, 90, 180 and 365 days, a custom integer
     (1–365), and "use default", which clears `[refresh:: N]`.
   - It writes through `api.freshness.setRefreshLine`, which also stamps.
   - Without the api, hide the row.
5. **Tests:**
   - Every listed gesture stamps an open Ready or Next task.
   - The stamp lands before the suffix, including after a priority or scheduled edit on
     a line that already has `[fresh::]`.
   - Cancel doesn't stamp.
   - Recurring tasks are never stamped.
   - A missing api leaves the lines as they were.
   - The refresh row in all three modes, including "use default".
   - Moved tasks are stamped at their destination.
   - `rg -n "upsertBulletProperty\([^)]*fresh"` finds nothing.
6. **Finish:** README row, a minor version bump, and a deploy.

## Phase: cycler-link-stamps — Status cycling and Task Link gestures stamp freshness

This phase works in bob-plugins `plugins/task-status-cycler/main.js` and
`plugins/block-id-prompt/main.js`.

1. **task-status-cycler.**
   - **What stamps:**
     - Alt+[ / Alt+] when the result is an open status (`[ ]`, `[*]`, `[/]`, `[?]`),
       including `applyBlockedStatusRetirementInEditor` (leaving Blocked by hand);
     - the counted range (`cycleTaskStatusRange`);
     - transcluded targets (`replaceResolvedTranscludedTaskLine`);
     - Ctrl+Enter reopening a done task.
   - **The Tasks-command catch.** `setActiveCheckboxStatus` first runs the Tasks command
     `set-status-symbol-to-*`, which rewrites the line itself. Choose one approach and
     justify it in the phase notes:
     - **(a)** For open-to-open transitions that need a stamp, use the existing local
       path (`setActiveCheckboxStatusLocalWithTaskMetadata`) so status and stamp are one
       edit. Confirm that Tasks' command adds nothing for these transitions.
     - **(b)** After the Tasks command resolves, apply the stamp as one follow-up
       single-line transaction. Only do it when the line still holds the same task with
       the expected new status.

     Prefer (a) if it keeps one undo step.

   - **Never stamp:** closing, Ctrl+Shift+] promotion, the dependency-ID normalizer,
     `recoverBlockedDependentsNow`.
   - Keep task-status-cycler's own api at version 1 unless you add a method.

2. **block-id-prompt.**
   - Add a `stampLine` option to `planTargetTaskUpdate`, applied last when it already
     rewrites the line (a status change, a retired schedule, or a new block ID).
   - Pass it from `applyPomodoroTaskLink` and from the `^^` paths
     (`completeTaskLinkWithExistingId`, `submitLinkTaskBlockId`).
   - Unlink, Task Link removal and Ctrl+6 never stamp.
3. **Both plugins:**
   - Get the stamper from ledger-tools api `version >= 3`, with the same guard style as
     `planBudgetNoticeSuffix` (typeof checks, reject Promise results, never throw).
   - Add a source comment explaining the placement exception.
4. **Tests** (`scripts/test-block-id-prompt.cjs`, the task-status-cycler test file):
   - each stamping and non-stamping gesture;
   - stamp placement on real-vault-shaped lines (P4, P5, P6);
   - a missing api;
   - one undo step, or the chosen alternative's guarantee.
5. **Finish:** README rows, minor version bumps, and deploys for both plugins.

## Phase: vault-review — Review note, dash chip, vim maps, chores, and config

1. **`~/bob/freshness.md`** (new).
   - Frontmatter: `parent: "[[gtd]]"` and `aliases: [Review, Freshness review]`.
   - An H1 "Freshness review".
   - A small `dataviewjs` counts line from `api.freshness.counts()`, showing "–" without
     the api.
   - A compact key legend (the review-outcomes table).
   - Then the block below, writing out `API` in full. A bare `app.` breaks headless
     queries.

     ````markdown
     ```tasks
     not done
     filter by function globalThis.app?.plugins?.plugins?.["bob-ledger-tools"]?.api?.freshness?.isDue?.(task) === true
     sort by function globalThis.app?.plugins?.plugins?.["bob-ledger-tools"]?.api?.freshness?.rank?.(task) ?? 0
     group by function globalThis.app?.plugins?.plugins?.["bob-ledger-tools"]?.api?.freshness?.tier?.(task) ?? "?"
     short mode
     hide toolbar
     ```
     ````

   - Check it headlessly with `bob query` (it must parse and be empty). Record the
     ordering that actually applies.

2. **`~/bob/dash.md` chip bar.**
   - Add a `review` chip before PLAN: `REVIEW <due> · <new> new · ✓ <today>`.
     - It is `external`, targeting `freshness`, with an `aria-label` that spells the
       counts out.
     - It gets a `.task-count-review` accent, plus `.task-count-over` styling while
       `new > 0`.
     - It shows "–" without `api.freshness`.
   - The sections are unchanged.
3. **`~/bob/obsidian_vimrc.md`**, following the existing `exmap` / `nmap` style:

   ```text
   exmap bob_next_due obcommand bob-navigation-hotkeys:jump-to-next-due-task
   exmap bob_prev_due obcommand bob-navigation-hotkeys:jump-to-prev-due-task
   nmap ]s :bob_next_due<CR>
   nmap [s :bob_prev_due<CR>
   ```

4. **`~/bob/gtd_daily.md`.** Edit the text in place and keep every field.
   - **Morning review** becomes:

     `Morning review (≈10 min): [[freshness|REVIEW]] until 0 new (]s, Alt+Shift+F), then clear what's due; [[dash#PENDING Tasks|PENDING]] → [[dash#NEXT Tasks|NEXT]]; link today's work, release the rest with Alt+N; ≤3 themes, highlight first`

   - **Weekly prune** gains:

     `; clear leftover [[freshness|REVIEW]] or lengthen that note's task_refresh; look for projects with no Next or Ready task`

5. **Config.**
   - Open chezmoi. In `home/dot_config/bob/config.yml`, add:

     ```yaml
     freshness:
       interval: 7 # days before a confirmed Ready task is due for review (docs/freshness.md)
       # stale_daily_budget: 15 # optional daily goal meter; never hides tasks
     ```

   - Commit per chezmoi's conventions, apply only that target, and confirm with
     `bob freshness list -f json | jq .config`.

6. **Sync** with `bob vault-sync run` and `bob vault-sync status`. Visual checks are on
   Bryan's checklist. Fill the `freshness.md` and `dash.md` Surfaces rows in bob-cli's
   `docs/freshness.md`.

## Phase: rollout — Install, end-to-end check, glossary term, and Bryan's checklist

1. **Install and deploy.**
   - From an up-to-date bob-cli master, run `cargo install --path . --locked --force`.
   - Pull the opened bob-plugins checkout to origin master and run
     `bob plugins sync -n -r "<opened bob-plugins path>"`.
   - Confirm that the deployed manifest versions match.
2. **End-to-end checks on apollo.** Paste the key output into the phase notes:
   - `bob freshness list` and `bob freshness list -f json | jq .counts`;
   - a `bob capture --dry-run -f json` Ensure Next on a Ready task. Its `task_line` must
     carry `[fresh:: <today>]` right before `[created::`;
   - `bob task-status-hooks --dry-run -f json` (no errors; Blocked counts unchanged from
     before the epic's seed);
   - the dash's four sections and `freshness.md` via `bob query`;
   - `rg -n "fresh::" ~/bob --glob '!.git'`, sampling 10 lines to confirm the placement.
3. **Memory.** Use `/sase_memory_write` (Edit And Republish) to add the glossary strand
   `sase/memory/glossary/task-freshness.md`, mirroring the existing strands'
   frontmatter:
   - `keyword: Task Freshness`, with `aliases: ["freshness"]`.
   - Body, about 120 words, adapted from the report's draft:
     - the local date a human last confirmed that an open task still needs doing as
       written, stored as `[fresh:: YYYY-MM-DD]` before the task's trailing Tasks
       fields;
     - supported Bob keymaps, Alt+F and `bob capture` edits of existing tasks stamp it;
       creation and automation never do;
     - a visible, non-recurring Ready task is due for review when it is new (no stamp),
       resurfaced (its `scheduled` date arrived after the stamp) or stale (the stamp is
       at least its interval old);
     - the interval comes from `[refresh:: N]`, then the note's `task_refresh`, then
       `freshness.interval` (7 days);
     - freshness never changes a lane, Today, schedule or priority;
     - it is not Zorg `@FRESHNESS`, `bob vault-sync` freshness, or P-level scheduling
       windows.

     Link `[[Task Link]]` only if the text names it.

   - Then run `sase memory init`, and confirm the roster lists "Task Freshness
     (freshness)".

4. **Finish** the `docs/freshness.md` Surfaces table and run `bob vault-sync run`.
5. **Bryan's checklist** goes in the final response; these steps are not automated.
   - **In Obsidian:**
     - The status bar shows `⟳ N due · N new · ✓ N today`.
     - `]s` / `[s` and Ctrl+Alt+J/K walk the queue across notes.
     - Alt+F stamps with no other change.
     - Alt+Shift+F stamps and advances.
     - `freshness.md` groups NEW → DUE.
     - The REVIEW chip links to it.
     - `fresh` shows as a muted pill.
     - The status bar counts match `bob freshness list`.
   - **Tune the load**, one change at a time:
     - Optionally triage `gkeep_inbox.md` once. Then set `task_refresh: 2` on
       `gkeep_inbox.md` and `mac_inbox.md`, or a long interval if it stays a someday
       list.
     - Consider `task_refresh: 14` on `sase.md`.
     - Set `freshness.stale_daily_budget` only if mornings run long.
   - **MacBook and athena:** reinstall `bob` so that capture stamps from the Mac, and
     run `bob plugins sync`.
   - **Trial:** two weeks. Keep the design if, on at least 10 of 14 mornings:
     - NEW reaches 0 before planning;
     - the whole ritual takes ≤10 minutes;
     - due debt is flat or falling;
     - no capture was missed;
     - no freshness write changed a lane, schedule, priority or Task Link.

## Choices this plan made that Bryan can overturn at review

1. **Scope is Ready only**, as requested. Next and Pending are already walked every
   morning. Because the predicate is lane-parametric, adding them later is a one-line
   change.
2. **Creation never stamps**, including desk captures, routed `@project` captures and
   hand-typed tasks. That is what "glance at every task I captured" requires.
3. **Automatic Blocked → Ready returns never stamp.** Short deferrals come back as
   RESURFACED instead.
4. **Names:** `fresh`, `refresh`, `task_refresh`, `freshness:`, `freshness.md`, `]s` /
   `[s`, Ctrl+Alt+J/K, Alt+F, and Alt+Shift+F (refresh and advance).
5. **`task_refresh` applies by residence** (the note that contains the task). It is not
   inherited through `parent` links.
6. **No daily cap or filter.** NEW is never capped. `stale_daily_budget` is an optional
   meter, off by default.
7. **The `seed` phase applies the cutover seed to the live vault**, dry run first and
   guarded by the parse-invariance check. The per-note intervals and the gkeep_inbox
   triage stay Bryan's.
8. **No decision record in this plan.** The report recommends one ("Freshness Is Stamped
   Only By Human Gestures And Read At Review Time"). Adding memory you didn't ask for
   needs your explicit OK, so it is left out. Say so in review feedback and it will be
   added to `rollout`.

## Deliberately not doing

- **A REVIEW section on the dash.** It would break the exclusivity decision; the dash
  gets a chip and the review note instead.
- **Auto back-off** (doubling the interval on unchanged refreshes), random sampling, a
  `never` interval (the maximum is 365), and auto-expiring stale tasks.
- **Hooks changes:** no stamping, clearing or linting in `bob task-status-hooks`, beyond
  tests that it preserves the field.
- **Teaching the Rust parsers `fresh`:** `bob query` must keep parsing lines exactly as
  Obsidian Tasks does.
- **A `bob plan`, tmux or hooks REVIEW meter**, and any change to Bob Mac Capture.
- **Emoji task-format support** for placement: the vault is Dataview-format, and
  `bob freshness` refuses other formats.
- **Stamping on view**, and a dependency-unblock tickler. Dependency returns come back
  STALE soon enough; add a rule only if one is missed.

## Risks

- **Placement regressions.** These are mitigated four ways: one helper per language, the
  shared P vectors, parse-invariance tests in Rust, and the seed's abort-on-change guard
  with a vault-wide before/after diff.
- **Load.** Expect 30–45 glances a morning at first, about 5–8 minutes. The levers come
  in order: longer note intervals, then deferring or cancelling during review, then the
  budget meter.
- **Rubber-stamping.** Refresh means "confirm", not "dismiss". Word the commands and
  Notices that way, and keep the other outcomes one key away.
- **Two evaluators drift.** `docs/freshness.md` owns the rule, and both sides run its
  vectors, as with Today.
- **Tasks cache lag.** Jumps and Notices adjust for the keys just stamped, and never
  write to a guessed line.
- **Line-digest churn.** A stamp changes the line digest, so an earlier Mac picker
  `line:digest` ref can come back Stale. That only happens when a gesture rewrites the
  line anyway.
