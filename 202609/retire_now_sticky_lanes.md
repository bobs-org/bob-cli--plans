---
tier: epic
title: 'Retire #now: sticky Next/Pending lanes and a ledger-derived Today'
goal: 'Linking a task makes it Next, working it makes it Pending, and no unlink path
  (hooks, keymap, capture drop, hand deletion) ever lowers it; only an explicit one-key
  release returns it to Ready. Today is read from the ledger when the dash renders,
  the dash shows mutually exclusive TODAY / PENDING / NEXT / READY sections with soft
  caps, and #now is gone from bob-cli, bob-plugins, Bob Mac Capture, the vault, and
  memory.

  '
phases:
- id: cutover-pause
  title: Pause the MacBook's hooks cron before the first 2026-10-01 pass
  depends_on: []
  size: small
  description: 'cutover-pause: best-effort, backed-up pause of the Mac''s task-status-hooks
    crontab line so tomorrow''s first pass cannot demote the current [/] and [*] tasks.'
- id: hooks-sticky
  title: Sticky lanes in bob task-status-hooks, docs, and superseding decision records
  depends_on: []
  size: medium
  description: 'hooks-sticky: stop the hooks lowering Next and In Progress when links
    disappear (daily-note tasks keep clearing), verify with a simulated next-day dry
    run, update the hooks docs, and write the two superseding decision records.'
- id: hooks-resume
  title: Install sticky hooks on the MacBook and restore the cron
  depends_on:
  - cutover-pause
  - hooks-sticky
  size: small
  description: 'hooks-resume: reinstall bob on the Mac from master, confirm with a
    dry run that lanes are kept, and restore the paused cron line byte-for-byte.'
- id: capture-toggle-lanes
  title: No bob capture path lowers a lane
  depends_on:
  - hooks-sticky
  size: medium
  description: 'capture-toggle-lanes: make @route+id! a link-presence toggle, keep
    In Progress under Ensure Next, word dropped rows as ''stays <status>'', and fix
    capture docs that promise hooks demotion.'
- id: today-core
  title: Define Today once; NEXT/PENDING lanes replace NOW in bob plan and the hooks
  depends_on:
  - hooks-sticky
  size: medium
  description: 'today-core: Rust Today engine with conformance vectors in docs/plan.md,
    lane meters and caps replacing NOW, bob plan JSON schema 2 with today_tasks, and
    the hooks plan_budget line.'
- id: capture-now-removal
  title: Remove
  depends_on:
  - capture-toggle-lanes
  - today-core
  size: medium
  description: 'capture-now-removal: delete the #now grammar, spans, completion context,
    picker group, now fields, and budget hint clause, then sweep bob-cli docs and
    README.'
- id: ledger-today-api
  title: bob-ledger-tools api v2 with a synchronous Today, lane budgets, and query
    refresh
  depends_on:
  - today-core
  size: medium
  description: 'ledger-today-api: synchronous Today cache with midnight rollover and
    the Tasks reload event, api v2 (isToday, todayRank, nextBudget, pendingBudget),
    lane chips in the bob-plan block, JS tests on the shared vectors.'
- id: link-toggle
  title: Ctrl+Shift+Enter toggles on link presence and never changes the lane
  depends_on:
  - hooks-sticky
  size: medium
  description: 'link-toggle: block-id-prompt links or unlinks by link presence, never
    writes the checkbox on unlink, offers the Work Log prompt for In Progress, and
    never lowers In Progress when linking.'
- id: release-key
  title: 'Alt+N commits or releases a lane; #now leaves Bob Navigation Hotkeys'
  depends_on:
  - ledger-today-api
  - link-toggle
  size: medium
  description: 'release-key: retarget Alt+N and the Ctrl+Shift+P pinned row from the
    #now toggle to commit (Ready to Next) and release (Next/In Progress to Ready,
    unlinking today), and delete every #now path.'
- id: dash-lanes
  title: Mutually exclusive dash sections, GTD chores, and lane caps config
  depends_on:
  - ledger-today-api
  - link-toggle
  - release-key
  size: small
  description: 'dash-lanes: rebuild dash.md as TODAY / PENDING / NEXT / READY with
    matching chips, swap the gtd_daily chores, and set max_next / max_pending in the
    chezmoi config.'
- id: mac-lanes
  title: 'Bob Mac Capture drops #now and presents the link-presence toggle'
  depends_on:
  - capture-toggle-lanes
  - capture-now-removal
  size: medium
  description: 'mac-lanes: remove now_tag, NOW badges, the now picker section, and
    ''stays in NOW''; present link/unlink toggles and unchanged statuses; regenerate
    fixtures; green macOS CI.'
- id: rollout
  title: Install, deploy, end-to-end check, and Bryan's checklist
  depends_on:
  - hooks-resume
  - capture-now-removal
  - dash-lanes
  - mac-lanes
  size: small
  description: 'rollout: install bob on apollo, sync every plugin, finish any Mac
    step hooks-resume could not, run the end-to-end checks, finalize the Surfaces
    table, and hand Bryan the triage and trial checklist.'
proposed_by: bbugyi200.apollo.3n
create_time: 2026-09-30 16:41:58
status: done
bead_id: bob-cli-2y
---

- **PROMPT:** [prompts/202609/retire_now_sticky_lanes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/retire_now_sticky_lanes.md)
- **BEAD:** [bob-cli-2y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2y/README.md)

# Plan: Retire `#now` — sticky Next/Pending lanes and a ledger-derived Today

> **⚠️ Time-sensitive (report, "Cutover → Tonight").** The currently installed
> `bob task-status-hooks` will demote about **48 `[/]` and 25 `[*]`** tasks to Ready on
> its first pass after the 2026-10-01 daily note exists (around 06:00). Only the
> MacBook's hand-installed crontab runs the hooks. Phase `cutover-pause` pauses that
> line as soon as this plan is approved; if the Mac is offline, Bryan should comment it
> out by hand (`docs/vault-git-sync.md`, "Mac scheduled maintenance"). If the pass has
> already run, treat the reset as the cutover triage and re-promote keepers from the
> report's appendix **after** `hooks-resume`. Until `link-toggle` is deployed,
> Ctrl+Shift+Enter still demotes on unlink: delete a link line by hand instead.

## Context

This epic implements the recommendations of
`research:202609/retire_now_sticky_lanes_ledger_today/retire_now_sticky_lanes_ledger_today.md`
("the report"): requirements R1–R13, its "How to implement Today", "Status rules and
gestures after the change", and "Cutover" sections. Every phase must read it first with
`sase artifact read`.

It reverses most of epic `bob-cli-2o` (`plan:202609/pomodoro_plan_budget_now_tag.md`),
which introduced `#now` on 2026-09-29. That plan is the best map of the `#now` surface
(phases plan-core, vault-now, capture-budget, close-drop, now-token, ledger-plan-view,
now-toggle, mac-close-now, rollout); read it when a phase below removes something it
added.

Bryan's request, as adjusted by the report (adjustments adopted in full):

- Stop the hooks wiping Next `[*]` and In Progress `[/]` when a Task Link disappears.
  The Next lane takes over what `#now` tracked, and Pending (`[/]`) is where
  long-running work (an agent swarm) waits after its link is removed from the daily
  note.
- Today comes from the ledger at read time. **Not** a file-path filter (it selects tasks
  that live in the daily note: 1 result against 7 live links) and **not** a
  hooks-managed `#today` tag (a second copy of the ledger rewritten on task lines every
  15 minutes).
- Today means dedicated links under today's **open** Pomodoros only; closed 🍅 lines are
  history, so a task worked this morning can still be parked.
- **Every** unlink path keeps the lane, not only the hooks; release is a separate key.
- The dash becomes mutually exclusive TODAY → PENDING → NEXT → READY (READY also
  excludes Today). The morning review walks PENDING → NEXT, then READY if there is room.
- A blocked or deferred task comes back as Ready (re-triage); this is documented, not
  hidden.

Repositories and surfaces:

| Surface          | Where                                                     | How to open                                                                                                                                                    |
| ---------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob` CLI        | this repo (`bob-cli`)                                     | your workspace                                                                                                                                                 |
| Obsidian plugins | `bob-plugins`                                             | `sase repo open bob-plugins -r "<why>"`, falling back to `gh:bobs-org/bob-plugins`. Read its `AGENTS.md`.                                                      |
| Mac capture app  | `bob-mac-capture`                                         | `sase repo open bob-mac-capture -r "<why>"`; the linked checkout is missing on apollo, so fall back to `sase repo open gh:bobs-org/bob-mac-capture -r "<why>"` |
| Vault            | `~/bob` (git-synced by `bob-vault-sync.service`)          | edit in place; never commit the vault by hand; finish with `bob vault-sync run`                                                                                |
| Config           | chezmoi source for `~/.config/bob/config.yml`             | `sase repo open chezmoi -r "<why>"`                                                                                                                            |
| MacBook          | `ssh mac` (user `bbugyi`; offline unless its lid is open) | best-effort only                                                                                                                                               |

## Design

### One marker, one job

| Marker                                                        | Owns                                             | Written by                                           |
| ------------------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| The ledger: dedicated Task Links under today's open Pomodoros | **Today**                                        | Bryan, capture, keymaps (links)                      |
| The checkbox: Ready `[ ]` → Next `[*]` → Pending `[/]`        | **Lane**                                         | link (raise), `=x` (Pending), Alt+N (commit/release) |
| `dependsOn` and future `scheduled`                            | **Blocked** `[?]`, derived, overrides every lane | `bob task-status-hooks`                              |

### Lane rules (authoritative for the hooks and every gesture)

1. Adding a Task Link under today's open Pomodoros raises Ready `[ ]` or Blocked `[?]`
   to Next `[*]`. Next and Pending are unchanged. Hooks promotion, including
   transcluded-dependency promotion, is otherwise unchanged.
2. An `=x` in-progress outcome sets `[/]` (unchanged).
3. Removing a link **never** changes the lane: the hooks, Ctrl+Shift+Enter,
   `@route+id!`, `=x~K`, `=~K`, and hand deletion.
4. Only **release** (Alt+N, or the Ctrl+Shift+P lane row) lowers Next or Pending to
   Ready. Releasing also removes the task's live links from today's open Pomodoros;
   releasing a Pending task offers the optional Work Log prompt.
5. Alt+N on a Ready task **commits** it to Next without linking it.
6. Blocked stays derived, overrides every lane, and recovers to the existing derived
   rank (Ready unless linked today or reached by recent activity).
7. **Carve-out (R9):** tasks that live in canonical daily notes (such as `^gtd`) keep
   today's derived clearing of `[*]`, including the recent-activity `KeptNext` grace.
8. **R12:** the `[/]` symbol, its "In Progress" name and IN_PROGRESS type stay. Only
   dash and chip labels say PENDING. Picker group names (`in_progress`) stay.

Gesture table (the report's, adopted):

| Gesture                                                                                                                                               | Today                      | Lane                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------- |
| **Link**: Ctrl+Shift+Enter on a task not linked today, `^^`, capture `@route:id` / `^route:id` / `@route+id!` on an unlinked task, the Mac `^` picker | joins                      | Ready or Blocked → Next; Next and Pending unchanged           |
| **Unlink**: Ctrl+Shift+Enter on a linked task or on a link line, `@route+id!` on a linked task, `=x~K`, start `~K`, hand delete                       | leaves                     | **unchanged**; for `[/]` in Obsidian, offer the Work Log note |
| `=x` in progress / deferred / complete                                                                                                                | carried / carried / leaves | → `[/]` / unchanged / done                                    |
| **Release**: Alt+N or the Ctrl+Shift+P lane row on Next/Pending                                                                                       | leaves if linked           | `[*]` → `[ ]`; `[/]` → `[ ]` via the optional Work Log prompt |
| **Commit**: Alt+N or the lane row on Ready                                                                                                            | none                       | `[ ]` → `[*]`                                                 |
| Defer (P-level or date) / Cancel                                                                                                                      | pruned (unchanged)         | → `[?]`, returns as Ready / `[-]`                             |

### Today (the definition `today-core` writes into `docs/plan.md`)

1. **Ledger and open entries** are exactly the plan budget's: today's daily note
   (`YYYY/YYYYMMDD.md`), its `## Pomodoros…` section, column-0 checkbox entries; an
   entry is open unless its status is `x`, `X`, or `-`.
2. **Every open entry counts**, including exempt ones (GTD) and unnamed placeholders.
3. **Only dedicated Task Links count:** a direct child bullet, at the entry's first
   child indentation, whose body after stripping 🍅 markers is exactly one plain
   `[[target#^id]]` or embedded `![[target#^id]]` block link, not struck through and not
   fenced. Deeper descendants and mixed-text bullets do not count. This is exactly
   `list_queued_links` in `src/native/capture_pomodoro_start.rs` (the `=x` / start
   lineup rule): reuse it, and pin its alias behavior in a vector.
4. **Resolution:** an empty target is the daily note itself; otherwise the hooks' rules
   apply (exact vault-relative path with or without `.md`, then a unique
   case-insensitive basename). Unresolved or ambiguous links are skipped; Rust reports
   them as the lint `today_link_unresolved`. JavaScript uses
   `metadataCache.getFirstLinkpathDest(target, dailyPath)`; ambiguous basenames are a
   documented divergence, and no conformance vector uses one.
5. **Today's tasks** are the resolved `#task` lines with that block ID whose status is
   open (Ready, Next, In Progress, or Blocked); done and cancelled tasks drop out.
   Deduplicate by (path, block ID), keeping the ledger order of first occurrence. The
   key is `"<vault path with .md>#<block id>"`.
6. Transcluded dependencies do **not** inherit Today; the hooks still promote them to
   Next.

### Lanes and caps

- **NEXT** is every `[*]` task and **PENDING** every `[/]` task that the dash's defaults
  show: not done, not dependency-blocked, not `#hide`, not under `_templates`, not under
  `_conflicts`, and no scheduled date after today. Each counts the **whole lane, Today
  included**, so counts don't swing during the day. Mirror the old `NOW_QUERY` defaults
  in `src/native/dataview/tasks/mod.rs` exactly, replacing only the tag test with the
  status test.
- **Config** (`plan:` in `~/.config/bob/config.yml`): `max_next` (default 15) and
  `max_pending` (default 10) replace `max_now`. A config that still has `max_now` loads
  without error, because unknown keys are ignored; test this in Rust and JS.
- **Lints** `next_cap_exceeded` (e.g. "NEXT has 16/15 tasks; release some with Alt+N")
  and `pending_cap_exceeded`. They never change the plan `status` and nothing is ever
  refused.
- **Visual language:** `TODAY n`, `PENDING n/10`, `NEXT n/15`, red when over (reverse
  video in tmux is unchanged; tmux shows no lanes).

### Contracts

- **`bob plan -f json`** moves to `schema_version: 2`: `now`, `caps.max_now` and
  `now_cap_exceeded` are removed; `today`, `today_tasks`, `next`, `pending`,
  `caps.max_next` and `caps.max_pending` are added. It is two days old and has no known
  consumer besides agents, so the bump is the honest signal.
- **`bob task-status-hooks` JSON:** `plan_budget` embeds the new report. The
  `cleared_in_progress` key stays but is always `[]`; `cleared` only lists daily-note
  tasks.
- **Capture JSON** stays `schema_version: 1`. The `now` fields, the `now_tag`
  span/need/context and the `now` picker group are removed; the Mac app already decodes
  all of them optionally (unknown span → neutral, unknown context → generic row, unknown
  group → `note`). The two-way toggle's `toggle_direction` becomes `"link"` or
  `"unlink"`; an older Mac app falls back to its generic success view until `mac-lanes`
  ships.
- **`bob-ledger-tools` api** becomes version 2: `nowBudget` is removed. Callers already
  use optional chaining, so a deployed older dash or plugin degrades to its fallbacks
  rather than failing.

## Cross-cutting rules for every phase

- **Read first:** the report (`sase artifact read`), plus the sections of
  `plan:202609/pomodoro_plan_budget_now_tag.md` for anything you remove.
- **New or changed CLI surface:** read `sase/memory/cli_rules.md` with
  `/sase_memory_read` (alphabetical options, short aliases, excellent `--help`, color on
  a TTY only).
- **bob-cli:** `just all` (fmt, lint, test) must pass. Keep touched Rust files under
  about 1500 lines where practical. Update `README.md` and `docs/*.md` in the phase that
  changes behavior.
- **bob-plugins:** `npm test` and `npm run validate` must pass. Bump the touched
  plugin's `manifest.json` minor version and update its README row. Deploy with
  `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>` (the default repo path
  may not exist on this host).
- **bob-mac-capture:** no Swift toolchain on this host, so macOS CI is the gate. Follow
  the repo's commit style (`feat(capture): …`) and push flow; the phase is not done
  until CI is green for your commit. Regenerate fixtures from a real `bob` built from
  bob-cli master.
- **Vault:** edit in place, keep each file's formatting, never touch daily notes or
  `done/` history, and finish with `bob vault-sync run` plus `bob vault-sync status`.
- **Memory:** only the files this plan names, through `/sase_memory_write`'s Edit And
  Republish path, then `sase memory init`. This plan's approval is the authorization.
- **MacBook steps** are best-effort: `ssh -o ConnectTimeout=15 mac …`. When it is
  unreachable, record exactly what is left in the phase notes and finish; `rollout`
  retries and Bryan's checklist covers the rest.
- **Epic workers** record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.

## Phase: cutover-pause — pause the MacBook's hooks cron before the first 2026-10-01 pass

Only the MacBook runs `bob task-status-hooks`
(`10,25,40,55 * * * * ~/.cargo/bin/bob task-status-hooks --retry-timeout 120 >> /var/tmp/bob_task_status_hooks.log`,
hand-installed, not chezmoi-managed). athena's `bob nightly` and apollo run no hooks.

1. Check reachability with `ssh -o ConnectTimeout=15 mac true`, retrying for about 10
   minutes. If the Mac stays unreachable, record "Mac unreachable; hooks cron NOT
   paused" and finish successfully.
2. Follow the change procedure in `docs/vault-git-sync.md` ("Mac scheduled
   maintenance"):
   - Save the live crontab twice: on apollo as `/tmp/mac-crontab-backup-<timestamp>.txt`
     and on the Mac as `~/mac-crontab-backup-<timestamp>.txt`.
   - Comment out **only** the `task-status-hooks` line by prefixing
     `#PAUSED retire-now cutover: `. Leave the highlights and projects jobs and every
     other line byte-identical.
   - Install with `ssh mac 'crontab -' < <edited copy>` and confirm with
     `ssh mac crontab -l`.
3. Record in the phase notes: both backup paths, the exact original line, the pause
   time, and `ssh mac 'tail -n 40 /var/tmp/bob_task_status_hooks.log'`. Say plainly
   whether a pass after the 2026-10-01 daily note already demoted tasks; if so, Bryan
   re-promotes keepers after `hooks-resume` (rollout checklist).
4. Change nothing else on the Mac: no vault edits, LaunchAgents, or installs.

## Phase: hooks-sticky — sticky lanes in `bob task-status-hooks`, docs, and superseding decision records

**Code** (`src/native/task_status_hooks/sync.rs`, `task_transition` and its caller):

- A Next `[*]` task with no desired status becomes `Unchanged`, **unless** it lives in a
  canonical daily note or the current ledger (the existing `is_daily_note`). There the
  current behavior stays: `KeptNext` when directly recent, else `Clear`. Pass that fact
  into `task_transition`. Don't count sticky non-daily tasks as `kept_next`.
- Remove the area/project In Progress rollback: `clear_stale_in_progress`,
  `Transition::ClearInProgress`, and any plumbing only it used. Keep
  `SyncResult.cleared_in_progress` serialized as an always-empty list, documented as
  kept for compatibility.
- Keep everything else: promotion and dependency propagation, derived Blocked and
  `Unblock` recovery ranks (today's links plus the previous daily's recent activity),
  duplicate, cancelled and completed-link cleanup, 🍅 repair, and the previous-daily
  lookback (it still feeds recovery ranks and the daily-note grace).

**Tests.** Update `src/native/task_status_hooks/tests/sync.rs` and
`tests/cli/task_status_hooks/` so non-daily clearing expectations become "stays", and
add cases for:

1. an unlinked `[*]` in an ordinary note stays `[*]`;
2. an unlinked `[*]` in an area or project note stays `[*]`;
3. an area/project `[/]` with no recent activity stays `[/]`;
4. a `^gtd`-style `[*]` in the daily note still clears once its GTD entry is closed and
   the grace does not apply;
5. Blocked still overrides `[*]` and `[/]`, and recovers to Ready when not linked;
6. Ready → Next on link is unchanged;
7. repeated runs stay idempotent.

**Verification (read-only).** Reproduce the report's simulated next-day pass: copy
today's daily note to a temp directory as `<tomorrow>.md`, keeping only open Pomodoro
entries, then run both the installed `bob` and the new build with
`BOB_DIR=~/bob BOB_DAY_FILE=<temp file> BOB_NOW=<tomorrow> … task-status-hooks --dry-run -f json`.
The installed binary should show large `cleared` / `cleared_in_progress` counts (the
report saw 25 and 48). The new build must show `cleared` only for daily-note tasks,
`cleared_in_progress: []`, and otherwise the same `marked_next`, `marked_blocked`, and
`unblocked`. Paste the counts into the phase notes. Never run a live pass.

**Docs.** In `docs/task-status-hooks.md`:

- the intro bullets and the previous-daily paragraph (it no longer "can keep an
  area/project In Progress task active");
- "Rolling Recent Activity": delete the rollback conditions and say what recent activity
  still does;
- the Next clearing policy, now daily-note tasks only;
- "Output": the meanings of `cleared`, `cleared_in_progress` and `kept_next`.

Also update the task-status-hooks section of `README.md`. Leave the capture and projects
docs to the capture phases.

**Memory.** Use `/sase_memory_write` (Edit And Republish), then `sase memory init`.
Mirror the existing strands exactly:

- frontmatter: `keyword`, `aliases`, one- or two-line `summary`, and
  `metadata.status: accepted` / `decided: <today>`;
- body sections: Applies to / Claim / Why (with rejected alternatives) / Cost / Reopens
  when, plus evidence refs (the report, this epic's plan ref from your phase bead, and
  this phase's commit).

1. **New `sase/memory/decisions/task-lanes-are-sticky.md`**: "Next And Pending Are
   Sticky Lanes; Only Blocked Is Derived".
   - **Summary:** linking raises Ready to Next and an `=x` close sets In Progress
     (PENDING); no unlink, hooks run, or capture drop lowers them; only Alt+N release
     returns a task to Ready; Blocked stays derived.
   - **Claim:** lane rules 1–8 above, compressed.
   - **Why:**
     - sticky lanes protect dropped work by default, where `#now` did only if tagged
       before dropping;
     - the one bulk `#now` tagging covered exactly the 73 `[/]` + `[*]` tasks and was
       stripped the same day;
     - Bryan cancelled a separate Submitted status ("WIP status should fill this role");
     - swarm work needs a quiet place to wait;
     - Pending → Next → Ready is Kanban's pull order and matches capture's
       `queued > in_progress > next`.
   - **Rejected alternatives:**
     - keep `#now` with a derived `[*]`: it repeats Today;
     - a Submitted/Waiting status;
     - no promotion on link: that is the status-lock trap;
     - age-based Next decay: only if the trial fails, never for Pending;
     - a hidden pre-block status.
   - **Cost:**
     - no automatic forgetting: about 2–3 tasks a day of lane growth at September's
       rates, so caps, chips, lints, a daily review with release and a weekly prune are
       required;
     - `[/]` no longer means "worked recently";
     - lanes don't survive Blocked;
     - dependency-promoted Next is sticky too;
     - about 80 legacy statuses need one-time triage.
   - **Reopens when:**
     - the two-week trial from `dash-lanes` fails its keep rule;
     - lanes must survive Blocked (then Blocked becomes an overlay).
2. **New `sase/memory/decisions/today-is-read-from-the-ledger.md`**: "Today Is Read From
   The Ledger, Never Written To Tasks".
   - **Summary:** Today is the open tasks with a dedicated Task Link under today's open
     Pomodoros, computed at read time by `bob plan` and bob-ledger-tools; never a tag,
     task-line field, or file-path filter; `#now` is retired.
   - **Claim:**
     - the definition lives in `docs/plan.md`, with Rust `today_tasks` and ledger-tools
       `isToday` sharing conformance vectors;
     - the dash sections are exclusive;
     - `#now` is removed everywhere, and "committed but not today" is the Next lane.
   - **Why:** the path-filter result above; `#today` churn, races and staleness;
     `filter by function` can call the plugin api, and the Tasks reload event exists.
   - **Rejected alternatives:**
     - a path filter;
     - a `#today` tag (a stopgap only);
     - keeping `#now` through its trial;
     - one `bob-dashboard` block (the fallback).
   - **Cost:**
     - it relies on an internal Tasks event;
     - headless `bob query` sees an empty Today;
     - there are two implementations;
     - resolver divergence on ambiguous basenames;
     - capture can no longer mark "this week" on new text.
   - **Reopens when:** Tasks removes the event or `app` access, or the refresh proves
     unreliable.
3. **Mark `now-tag-is-user-owned`** `metadata.status: superseded`, `superseded_by` both
   new records. Add one back-link sentence saying `#now` is retired and the Next lane
   now holds this week's commitments.
4. **Mark `task-status-is-derived`** `metadata.status: superseded-in-part`,
   `superseded_by: [[decisions/task-lanes-are-sticky]]`. Add one back-link sentence:
   Next and In Progress are no longer derived; Blocked stays derived as stated.

Do not otherwise edit either old body.

## Phase: hooks-resume — install sticky hooks on the MacBook and restore the cron

1. Check reachability as in `cutover-pause`. If the Mac is unreachable, record it and
   finish; `rollout` retries.
2. Find how `bob` was installed, e.g. from `~/.cargo/.crates.toml` or `.crates2.json`.
   - **A `path+file://` install:** the Mac checkout must be clean and on `master`. Then
     `git pull --ff-only`, confirm HEAD contains `hooks-sticky`'s commit, and run
     `cargo install --path . --locked --force`.
   - **A `git+` install:** reinstall from the same source with `--locked --force`.
   - **Anything else** (a dirty checkout, another branch, a build failure): stop, leave
     the cron as it is, and record the exact steps Bryan must take.
3. Verify with `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json'`
   (summarize with `jq`): `cleared` lists only daily-note tasks and
   `cleared_in_progress` is empty.
4. If `cutover-pause` paused the line, restore the original line from its backup,
   install it, and confirm `crontab -l` matches the backup byte-for-byte.
5. Record the installed commit, the dry-run summary, and the final cron state.

## Phase: capture-toggle-lanes — no `bob capture` path lowers a lane

**`@route+block-id!` becomes a link-presence toggle**
(`src/native/capture/task_toggle.rs`, `src/native/capture_task_toggle.rs`):

| Linked under an open Pomodoro today? | Status                         | Result                                                                                |
| ------------------------------------ | ------------------------------ | ------------------------------------------------------------------------------------- |
| no                                   | Ready `[ ]` / Blocked `[?]`    | Next, with the link inserted under the implicit current/next open Pomodoro (as today) |
| no                                   | Next `[*]` / In Progress `[/]` | link inserted; status unchanged                                                       |
| yes                                  | any open status                | every matching link under every open Pomodoro removed; status unchanged               |
| —                                    | done, cancelled, unknown       | the existing error                                                                    |

- "Linked" means `plan_link_removal`'s matcher finds a link under an open entry of
  today's daily note.
- The link direction uses the existing `plan_task_link`: Ready/Blocked → Next; Next/In
  Progress keep their status; a single future `scheduled` is retired, with the Schedule
  Log rules.
- Delete the In Progress error ("needs a work summary"), and delete `plan_task_open` if
  nothing else uses it.
- The `dependsOn` warning fires only when the status became Next.
- JSON: `toggle_direction: "link" | "unlink"`, `status_changed`,
  `removed_pomodoro_links` for unlink, and the existing insertion fields for link.
- Human output, for example: `linked ^fix-it under GOALS · set Next`,
  `linked ^x · stays In Progress`,
  `unlinked ^fix-it from 2 open Pomodoros · stays Next`.

**Ensure Next** (`@route+id`, `@route+id#pomodoro`, `src/native/capture/ensure_next.rs`)
keeps In Progress as In Progress, using `plan_task_link` semantics, with
`status_changed: false`. Ready, Blocked and Next behave as today.

**Dropped rows.** For `=x~K` and `=~K`, the human output replaces ` · stays in NOW` with
` · stays <status name>` on every dropped row (`capture/output.rs`,
`capture/start_output.rs`). Leave the `now` JSON fields for `capture-now-removal`.

**Docs and help.**

- `capture/cli.rs` help.
- `docs/capture.md`:
  - rename "Task status toggle" to "Task Link toggle" and rewrite its table and prose,
    including Ensure Next's eligible statuses;
  - the `=x` outcome table's dropped row becomes "none — the task keeps its lane";
  - "Dropping queued Task Links as a session starts";
  - every other sentence saying the next hooks run demotes Next → Ready.
- `README.md` capture rows.

**Tests.** Unit and CLI tests for:

- each table row, and Ensure Next on `[/]`;
- the removed In Progress error;
- the JSON shapes and human output;
- the dropped-row wording for close and start.

## Phase: today-core — define Today once; NEXT/PENDING lanes replace NOW in `bob plan` and the hooks

**Today engine** (new `src/native/plan_budget/today.rs`):

- `today_links(contents)` is pure. Walk open entries (`capture_pomodoros::scan`) and
  call `list_queued_links` on each. Record the entry line and name, ledger line, target,
  block ID and `embedded`.
- `today_tasks(bob_dir, daily_relative, contents)` returns the rows plus lints. It
  resolves per rule 4, reads task lines with `note_tasks`, keeps open statuses, and
  deduplicates.
- Reuse an existing resolver (the hooks' note index / `resolve_task_reference`, or
  `vault_links`) rather than writing a third. A targeted lookup (exact path, then a walk
  for a unique basename) is acceptable if none fits without a full scan.

**Lanes.**

- Replace `NOW_QUERY` with `NEXT_QUERY` and `PENDING_QUERY`.
- Replace `count_now`/`NowBudget` with
  `count_lanes(bob_dir, today, &PlanConfig) -> Lanes { next, pending }`, each
  `{count, cap, over}`.
- Add the two lints and delete `LINT_NOW_CAP`.
- **Keep `has_now_tag`** in `plan_budget/mod.rs`: capture still uses it until
  `capture-now-removal`.

**Config** (`src/native/config/plan.rs`): add `max_next` (15) and `max_pending` (10),
remove `max_now`, and validate as before. Add a test that a config still containing
`max_now` loads.

**`bob plan`.** JSON `schema_version: 2`:

```json
{
  "ok": true,
  "schema_version": 2,
  "date": "2026-10-01",
  "daily_file": "2026/20261001.md",
  "caps": {
    "max_themes": 3,
    "max_links": 10,
    "max_next": 15,
    "max_pending": 10,
    "strict": false
  },
  "status": "ok",
  "themes": { "count": 3, "cap": 3, "over": false },
  "links": { "count": 7, "cap": 10, "over": false },
  "today": { "count": 7 },
  "next": { "count": 12, "cap": 15, "over": false },
  "pending": { "count": 8, "cap": 10, "over": false },
  "today_tasks": [
    {
      "path": "sase.md",
      "block_id": "fix-it",
      "line": 120,
      "status_symbol": "*",
      "status_name": "Next",
      "text": "Fix it",
      "entry_line": 55,
      "entry_name": "BOB",
      "ledger_line": 56
    }
  ],
  "entries": ["…unchanged…"],
  "warnings": ["…"]
}
```

Human output:

```text
bob plan · Thu 2026-10-01 · 2026/20261001.md

  PLAN  3/3 themes · 7/10 links      TODAY 7 · PENDING 8/10 · NEXT 12/15

  ★ GOALS    ▶ 0945-1015   3 links
    …

  TODAY
    [*] sase#^fix-it        Fix it
    [/] bob#^capture-stop   Better capture stop
```

- With no daily note, show `TODAY 0` and still show the lanes.
- Colors follow the existing meters.
- The about text in `plan_budget/cli.rs` and `src/runner.rs` becomes "Show today's plan
  budget, Today's tasks, and the NEXT/PENDING lanes". Options are unchanged.

**Hooks.**

- `plan_budget_for_sync` uses `count_lanes` plus the Today engine; it already has the
  daily contents.
- The human line becomes
  `plan 3/3 themes · 7/10 links · TODAY 7 · PENDING 8/10 · NEXT 12/15`, with over-cap
  meters red.
- Remove NOW.

**Docs.**

- `docs/plan.md`:
  - retitle it "Plan budget, Today, and lanes";
  - replace "NOW (this week's bets)" with a "Today" section (the rules above, verbatim)
    and "Lanes (NEXT and PENDING)";
  - update the config block, JSON, lint table, and "Surfaces" (fill `bob plan` and the
    hooks; mark the ledger-tools, dash and Notice rows for later phases).
- Add **Today conformance examples**. Each gives the ledger, the notes it resolves
  against, and the expected ordered keys:
  - **T1 GTD:** `- [ ] () — GTD` / `\t- [[#^gtd]]` → `<daily>#gtd`.
  - **T2 markers:** `🍅 [[a#^x]]` counts, `~~[[a#^y]]~~` doesn't, `![[a#^z]]` counts.
  - **T3 closed entries:** links under `[x]` and `[-]` entries don't count.
  - **T4 shapes:** a mixed bullet `Review [[a#^m]]` and a link nested under a note
    bullet don't count.
  - **T5 dedupe:** one task under two open entries, and `[[a#^x]]` plus `[[dir/a#^x]]`
    resolving to the same note → one key at its first position.
  - **T6 fenced:** a link inside a fenced block doesn't count.
  - **T7 status:** linked `[x]` and `[-]` tasks drop out; `[?]` stays.
  - **T8 unresolved:** `[[missing#^q]]` gives no key, plus `today_link_unresolved`.
  - **T9 alias:** `[[a#^x|alias]]`, pinned to whatever `list_queued_links` does.
- Also update:
  - the `docs/README.md` index line;
  - the NOW mentions in `docs/task-status-hooks.md` (human line, JSON example, the
    meters sentence);
  - `README.md` (command table, "Plan budget", and the Environment "15 NOW tasks").

**Tests.**

- A unit test per vector.
- A lane fixture vault covering:
  - open `[*]` and `[/]`;
  - done tasks;
  - `#hide`;
  - `_templates/`;
  - a future `scheduled` date;
  - dependency-blocked tasks.
- Rewrite `tests/cli/plan.rs` from NOW to lanes and Today, JSON and human (no ANSI when
  piped).
- Update the hooks plan-budget CLI test.
- The help tests must keep passing.

## Phase: capture-now-removal — remove `#now` from capture, pickers, close/start rows, and bob-cli docs

After this phase, a trailing `#now` after the route is the ordinary trailing-tag error
like any other `#tag`, and `#now` typed before the route is plain body text. Keep the
grammar tests where `#now` is a **Pomodoro-name selector** (`@dev+id#now!`); they are
unrelated.

Remove:

- **Grammar** (`src/native/capture_language/`):
  - `markers.rs`: `NOW_TAG`, `is_now_tag`, `is_now_tag_prefix`,
    `strip_trailing_now_tag`, the `#now` exemptions in `reject_legacy_bullet_markers`
    and its editor twin, and `now_tag_body_error`;
  - `line.rs`: the pop/re-append in `resolve_line`;
  - `item.rs` and `tokens.rs`: the `#now` usage errors;
  - `editor_parse.rs`: `now_tag`, `now_tag_without_task_diagnostic`,
    `parse_editor_now_tag_item`, the child and parent spans, `now_tag_only`, and
    `Need::NowTag`;
  - `editor_model.rs`: `SpanKind::NowTag` and `Need::NowTag`;
  - `completion.rs`: `CompletionContext::NowTag`.

  Also remove `now_tag` from the "Needs" list in the `capture_parse.rs` help.

- **capture-complete** (`capture_complete.rs`):
  - `NowTagCandidate` and `Candidates::NowTag`;
  - `now` on `TaskLinkCandidate` and `ActiveTaskCandidate`;
  - the human rows and tails, `context_label`, and the help text.
- **`^` picker** (`capture_active_tasks.rs`): stop listing Ready `#now` tasks, drop
  `ActiveTask.now`, and renumber the queue tiers.
- **`:` picker** (`capture_link_tasks.rs`): delete `LinkTaskGroup::Now`. Those tasks
  fall into `note`, so the order is `queued`, `in_progress`, `next`, `note`.
- **Close and start rows:**
  - `PomodoroCloseTask.now`, `StartTaskRow.now`, `PomodoroCloseTaskJson.now` and
    `PomodoroStartTaskJson.now`;
  - the code that sets them in `capture/pomodoro_close.rs` and
    `capture/pomodoro_start.rs`.
- **Budget hint** (`capture/budget.rs`, including the strict-refusal message): "queue it
  with ^ or defer with p:<N>".
- **Finally:** `has_now_tag` and any remaining NOW tests.

**Docs sweep.**

- `docs/capture.md`:
  - the grammar table rows and examples;
  - delete the `### This week's #now` section;
  - the picker groups, the start and close JSON notes, and the span-kind list;
  - completion contexts and `group` values.
- `docs/projects.md`: the cancel Notice now has no NOW chip and no "keeps `#now`"
  effect.
- Leftovers in `docs/plan.md`, and `README.md`.

**Final check.**
`rg -n '#now|now_tag|NowTag|max_now|now_cap|stays in NOW|This week.s bet' src tests docs README.md justfile`
must leave only intentional hits: Pomodoro-name selector tests, and any historical
sentence saying `#now` was retired. `just all` must pass.

## Phase: ledger-today-api — bob-ledger-tools api v2 with a synchronous Today, lane budgets, and query refresh

This is bob-plugins `plugins/bob-ledger-tools/` (`main.js`, `styles.css`) plus tests.

**Pure helpers**, exported through `module.exports.helpers`:

- `computeTodayLinks(content, dailyPath)`: rules 1–3, mirroring `list_queued_links`
  exactly.
- `resolveTodayKeys(links, dailyPath, resolve)`: ordered, unique keys. An empty target
  maps to `dailyPath`.
- `laneBudgetFromTasks(tasks, today, caps, lane)`: `"next"` is symbol `*`, `"pending"`
  is type IN_PROGRESS. It uses the visibility predicate `nowBudgetFromTasks` used.
- Delete `nowBudgetFromTasks`, `hasNowTag`, `PLAN_NOW_TAG_RE` and `PLAN_LINT_NOW_CAP`.
- Caps: `maxNext` (15) and `maxPending` (10) from `max_next`/`max_pending`; drop
  `maxNow`. Verify `coercePlanCaps` still accepts a config containing `max_now`.

**Synchronous Today cache**, `{date, dailyPath, keys, rank}`:

- **Build it:**
  - on layout ready, via `vault.cachedRead`;
  - on `metadataCache` `changed` for the daily path, using the event's content;
  - once on `resolved`;
  - on daily-note create, delete, or rename;
  - at local-midnight rollover (a one-minute `registerInterval` comparing
    `todayDailyPath(new Date())` is fine).
- Resolve targets with `app.metadataCache.getFirstLinkpathDest`.
- Never await inside the api. Before the first build, `isToday` returns `false`.
- **When the key set changes:**
  - call `app.workspace.trigger("obsidian-tasks-plugin:reload-open-search-results")`, so
    every open Tasks query re-reads (report, "What I verified", row 4);
  - re-render bob-plan blocks, debounced like the existing re-render.

  Keep the event name in a named constant, pin it with a test, and confirm it against
  the installed bundle in `~/bob/.obsidian/plugins/obsidian-tasks-plugin/main.js`.

**API:**
`this.api = Object.freeze({ version: 2, caps(), planBudget(opts), todayKeys(), isToday(task), todayRank(task), nextBudget(), pendingBudget() })`.

- `isToday(task)` uses `task.path` plus the block ID from the Tasks task. Tasks 8.4.0
  keeps it in `blockLink` as ` ^id`; confirm this in the bundle. Missing fields return
  `false`.
- `todayRank` returns the ledger index, or `Number.MAX_SAFE_INTEGER` for tasks that
  aren't Today.
- `todayKeys()` returns a copy.
- `nowBudget` is removed.

**bob-plan block.**

- Chips:
  - `PLAN t/3 · l/10`;
  - `TODAY n`, counting not-done cached tasks where `isToday` is true;
  - `PENDING n/10` → `dash#PENDING Tasks`;
  - `NEXT n/15` → `dash#NEXT Tasks`.
- The lane chips turn red when over.
- `next_cap_exceeded` and `pending_cap_exceeded` replace `now_cap_exceeded`.
- Styles: drop `.bob-plan-now` and add today, pending and next accents using the
  `--task-status-in-progress` and `--task-status-next` variables.

**Tests.** Use a new `scripts/test-ledger-tools-today.cjs` registered in `package.json`,
or extend the plan-budget test file. Cover:

- `docs/plan.md`'s Today vectors, verbatim, with a stub resolver (cite the doc);
- lane budgets;
- caps, including a legacy `max_now`;
- the api v2 shape;
- cache rebuilds on `changed` and on rollover;
- the refresh firing only when the keys change;
- the event-name pin.

**Finish.** Update the README's bob-plan paragraph and document api v2 as the supported
interface. Update the plugin's README row, bump its minor version, and deploy.

## Phase: link-toggle — Ctrl+Shift+Enter toggles on link presence and never changes the lane

This is bob-plugins `plugins/block-id-prompt/main.js`.

1. **Task mode** (`openPomodoroTaskLink`) decides on **link presence**, not the
   checkbox.
   - The task counts as linked when today's daily note (its live editor buffer when
     open) has a live, non-struck link to its block ID under any open Pomodoro. That is
     the set `planAllOpenPomodoroLinkCleanup` would remove.
   - Linked → unlink; otherwise → link. A task without a block ID is never linked, and a
     task linked only under a completed Pomodoro gets linked again.
2. **Unlink.**
   - Remove those links and never write the checkbox.
   - For `[/]`, first open the Work Log prompt, reusing `WorkSummaryPromptModal`:
     - retitle it "Unlink task", with a subtitle like "Optional: why is this pending?
       Saved to the Work Log." and the button "Unlink";
     - Escape cancels the whole gesture;
     - a blank submit unlinks without a log.
   - Replace `planTargetTaskOpenUpdate` here with a Work-Log-only plan, and delete it if
     it becomes unused.
3. **Link.** The flow is unchanged except for status.
   - Ready and Blocked become Next (`forceNext`, still retiring a unique future
     schedule).
   - Next and In Progress stay unchanged: fix `planTargetTaskUpdate` so `forceNext`
     never lowers `/`.
4. **Link mode** (cursor on any Task Link: `startTaskLinkOpen` / `applyTaskLinkOpen`).
   Delete the selected link and today's open-Pomodoro duplicates as today. Drop the
   `resetsStatus` status write. A `[/]` target gets the same optional prompt.
5. **Notices**, keeping the plan-budget suffix:
   - `Linked · Next`, `Linked · stays In Progress`;
   - `Unlinked · stays Next`, `Unlinked · stays In Progress · Work Log updated`;
   - `Task Link removed · stays Next`.
6. The command id, name, and Ctrl+Shift+Enter hotkey are unchanged.
7. **Tests** (`scripts/test-block-id-prompt.cjs`):
   - rewrite the Next→Open and `[/]` pause tests as lane-preserving unlink tests;
   - linking a `[*]` or `[/]` task;
   - linked only under a completed Pomodoro → link;
   - `forceNext` never lowers `/`;
   - link mode keeps `*`, `/`, ` ` and `?`;
   - a blank vs a nonblank summary;
   - Escape.
8. **Finish:** update the README row (the Ctrl+Shift+Enter text), bump the minor
   version, and deploy.
9. **Memory:** update the glossary's Work Log strand. Find its file with the memory
   tooling, and edit it through `/sase_memory_write`, then `sase memory init`. Its
   Ctrl+Shift+Enter sentence becomes: unlinking an In Progress task prepends its
   nonblank summary, a blank summary leaves the log untouched, and the task stays In
   Progress.

## Phase: release-key — Alt+N commits or releases a lane; `#now` leaves Bob Navigation Hotkeys

This is bob-plugins `plugins/bob-navigation-hotkeys/main.js`.

1. **The command.**
   - Replace `toggle-now-tag` with `toggle-task-lane`, "Commit to Next / release to
     Ready", default hotkey Alt+N.
   - Retarget the counted `N<Alt+N>` capture-phase Vim listener, and reuse the #now
     toggle's target discovery: a task line plus the next N tasks, or a dedicated Task
     Link plus the next N sibling links, resolved to their tasks.
   - If `~/bob/.obsidian/hotkeys.json` binds `bob-navigation-hotkeys:toggle-now-tag`,
     move the binding to the new id and run `bob vault-sync run`.
2. **Semantics** over the targets:
   - **Commit:** if any target is Ready, every Ready target becomes Next. Other targets
     are untouched, and no link is written.
   - **Release:** otherwise, every Next and In Progress target becomes Ready, and each
     released task's live links under today's open Pomodoros are removed (reuse
     `planDeferredPomodoroLinkCleanup`).
     - If any target is In Progress, first ask once for an optional Work Log summary,
       worded like link-toggle's prompt but titled "Release task".
     - A nonblank summary is prepended to each released In Progress task's Work Log; a
       blank one writes nothing; Escape cancels.
   - **Blocked** targets are skipped and counted in the Notice ("2 Blocked skipped —
     Blocked is derived"). Done, cancelled and non-task targets keep the existing
     refusals.
   - **Writes** follow the #now toggle's cross-note plan and rollback: targets first,
     then the daily note, refusing if any preimage changed.
   - **Work Log planning:** copy block-id-prompt's pure `planWorkLogInsertion`
     byte-for-byte, with a source comment. That is the repo's copy convention; plugins
     never import each other's `main.js`.
3. **The Ctrl+Shift+P pinned row.**
   - Replace the `#now` row, in the same slot (`order: -1`) in task and link mode, with
     a lane row whose detail reads `lane · commit to Next` or `lane · release to Ready`.
   - Releasing In Progress goes through an optional reason stage, like the cancel row's.
4. **Notices:**
   - `→ Next · 3 tasks · NEXT 13/15`;
   - `→ Ready · 2 tasks · unlinked 1 from today · NEXT 11/15 · PENDING 7/10`;
   - 🔴 plus "prune at the weekly review" when over.

   Read the counts from ledger-tools `api.nextBudget()` / `pendingBudget()`
   (`version >= 2`) before the write, adjust them by the change because the Tasks cache
   lags, and omit them when the api is unavailable.

5. **Delete every `#now` path:**
   - `hasNowTag`, `getNowTagTokenSpans`, `addNowTagToLine`, `removeNowTagFromLine`;
   - `planNowToggleBatch`, `buildNowToggleNotice`, `readNowBudgetValue`;
   - `describeNowToggleRow`, `applyNowToggleFromPicker`, `renderNowToggleItem`;
   - the cancel flow's `nowTaggedCount`, `nowChip`, `anyNow`, and its "keeps #now"
     effect.

   Defer paths (future date → Blocked plus prune) are unchanged.

6. **Tests.** Replace the now-toggle tests with lane tests:
   - commit, release, and mixed batches;
   - an In Progress summary that is blank, nonblank, or cancelled;
   - Blocked skipped;
   - link mode and counted mode;
   - the picker row in both modes;
   - Notices with and without api v2;
   - rollback when a preimage changed.

   Update the cancel tests that assert NOW chips.
   `rg -n '#now|nowBudget|NOW' plugins/bob-navigation-hotkeys scripts/test-navigation-hotkeys.cjs`
   must leave only unrelated hits.

7. **Finish:** update the README row (Alt+N), bump the minor version, and deploy.
8. **Memory:** add to the glossary's Work Log strand that Alt+N releasing an In Progress
   task prompts for the same optional summary. Use `/sase_memory_write`, then
   `sase memory init`.

## Phase: dash-lanes — mutually exclusive dash sections, GTD chores, and lane caps config

1. **`~/bob/dash.md` sections.** Keep the frontmatter `TQ_extra_instructions` as the
   note-wide defaults. Replace `### NOW Tasks`, `### WIP Tasks`, `### NEXT Tasks` and
   `### READY Tasks` with the blocks below. `API` stands for
   `globalThis.app?.plugins?.plugins?.["bob-ledger-tools"]?.api`; write it out in full,
   because a bare `app.` breaks headless queries.

   ````markdown
   ### TODAY Tasks

   ```tasks
   not done
   filter by function API?.isToday?.(task) === true
   sort by function API?.todayRank?.(task) ?? 0
   ```

   ### PENDING Tasks

   ```tasks
   status.type is IN_PROGRESS
   filter by function API?.isToday?.(task) !== true
   sort by created
   ```

   ### NEXT Tasks

   ```tasks
   status.name includes Next
   filter by function API?.isToday?.(task) !== true
   sort by priority
   ```

   ### READY Tasks

   ```tasks
   status.type is TODO
   filter by function API?.isToday?.(task) !== true
   ```
   ````

   - The three status sections are disjoint by type (IN_PROGRESS, ON_HOLD Next, TODO),
     and the lower three exclude Today.
   - Check each block headlessly with `bob query`: it must parse, TODAY must be empty,
     and the others unfiltered by Today.
   - Check whether the default `group by path` and `sort by function` lines override the
     section sorts, and record the ordering that actually applies.

2. **The chip bar** (the `dataviewjs` block):
   - Order: TODAY, PENDING, NEXT, READY, BLOCKED, PLAN. Each label must match its
     heading, because the chip href is `#<label> Tasks`.
   - **TODAY** counts active, not dependency-blocked tasks where `api.isToday(t)` is
     true, and shows "–" without the api.
   - **PENDING** and **NEXT** are the whole `[/]` and `[*]` lanes. They prefer
     `api.pendingBudget()` / `api.nextBudget()` for count, cap and over, and fall back
     to the inline count with caps 10 and 15.
   - **READY** counts TODO tasks that aren't Today.
   - Remove the NOW chip, `NOW_CAP`, and the `nowBudget` code.
   - Rename `.task-count-wip` to `.task-count-pending`, drop `.task-count-now`, and add
     a `.task-count-today` accent.
   - Keep the caps in the `aria-label`s.
3. **`~/bob/gtd_daily.md`.**
   - Cancel the open "Pick today: ≤3 themes from yesterday + [[dash#NOW Tasks|NOW]]…"
     chore and the "Weekly review: re-tag [[dash#NOW Tasks|NOW]]…" chore: set `[-]`,
     append `  [cancelled:: <today>]`, and keep their other fields.
   - Add, in the file's style:
     - `- [ ] #task Morning review (≤5 min): [[dash#PENDING Tasks|PENDING]] → [[dash#NEXT Tasks|NEXT]] → [[dash#READY Tasks|READY]]; link today's work, release the rest with Alt+N; ≤3 themes, highlight first  [repeat:: every day when done]  [created:: <today>]  [scheduled:: <today>]`
     - `- [?] #task Weekly prune: release [[dash#NEXT Tasks|NEXT]] to ≤15 and [[dash#PENDING Tasks|PENDING]] to ≤10 with Alt+N, promote from [[dash#READY Tasks|READY]], defer with P-levels, check hours and created vs closed  [repeat:: every week on Monday when done]  [created:: <today>]  [scheduled:: <next Monday>]`

       Use `[?]` because a future schedule derives Blocked.

4. **Config.**
   - Open chezmoi. In `home/dot_config/bob/config.yml`, replace the `max_now` line with
     `max_next: 15 # [*] Next lane soft cap (docs/plan.md)` and
     `max_pending: 10 # [/] In Progress lane soft cap, shown as PENDING`, and remove
     `#now` from the block comment.
   - Commit it per chezmoi's conventions and apply only that target.
   - Confirm with `bob plan -f json | jq .caps`.
5. **Sync.** Run `bob vault-sync run` and `bob vault-sync status`. Note that visual
   verification in Obsidian is on Bryan's checklist.

## Phase: mac-lanes — Bob Mac Capture drops `#now` and presents the link-presence toggle

This is bob-mac-capture. It has no `AGENTS.md`; follow the README's "Development"
section and the `justfile`.

1. **Remove:**
   - the `.nowTag` semantic category, its palette entry, and `now_tag` in
     `completionSpanKinds` / `completionNeeds`;
   - the `.nowTag` completion row (`star.circle`, "This week's bet");
   - NOW badges on active-task and task-link candidates, close rows, and start rows,
     including decoding of the `now` key on `PomodoroStartTask`, `PomodoroCloseTask` and
     `CaptureCompletionCandidate`;
   - the picker's `now` section ("This Week's Bets", `TaskLinkGroup.now`, the section
     kind and view switches);
   - the plan-cap hint's "#now" clause, which now matches bob's hint;
   - README mentions.

   Replace "stays in NOW" captions with "stays <status>" using the row's status.
   **Keep** the pink current-Pomodoro `NOW` pill (`section.isCurrent`); it is unrelated.

2. **Toggle presentation** (`CaptureTogglePresentation`):
   - add `link` and `unlink` directions;
   - the transition text uses `status_changed` for every behavior ("unchanged" when it
     is false), so Ensure Next on In Progress reads "unchanged";
   - unlink shows the removed-link count and "stays <status>";
   - keep decoding `next` and `open` for older bob.
3. **Fixtures.**
   - Delete `now-tag-parse.json`, `now-tag-parse-incomplete.json`,
     `now-tag-complete.json`, and their `fake-bob` branches.
   - Regenerate every fixture that carried `now`: `pomodoro-close-drop-now.json`, the
     `pomodoro-start-*` files, and `task-link-complete-*`.
   - Add `link` and `unlink` toggle fixtures, generated from a real `bob` built from
     bob-cli master after `capture-now-removal`.
4. **Finish:** update the tests and the README's runtime-contract section, commit/push,
   and get macOS CI green.

## Phase: rollout — install, deploy, end-to-end check, and Bryan's checklist

1. **Install.** From an up-to-date bob-cli master checkout, run
   `cargo install --path . --locked --force`. Then pull the opened bob-plugins checkout
   to origin master and run `bob plugins sync -n -r "<opened bob-plugins path>"`.
2. **Mac.** If `hooks-resume` couldn't finish, retry it now. Otherwise confirm the hooks
   line is active in `ssh mac crontab -l` and that the Mac's dry run keeps lanes.
3. **End-to-end on apollo.** Paste the key output into the phase notes:
   - `bob plan`
   - `bob plan -f json | jq '{schema_version, today, next, pending, caps, n: (.today_tasks | length)}'`
   - `bob task-status-hooks --dry-run -f json`, summarizing `cleared`,
     `cleared_in_progress` and `plan_budget`
   - `bob capture-parse -f json -- 'Fix it @sase^fix-it #now'`, which must give the
     ordinary trailing-tag error
   - `bob capture --dry-run -f json -- '@<route>+<id linked today>!'`, which must give
     `toggle_direction: "unlink"` and `status_changed: false`
   - the dash's four sections via `bob query`
   - `rg` for `#now|nowBudget|max_now|now_tag` across bob-cli, bob-plugins and the Mac
     repo, which must leave only intentional hits
4. **Finish.** Finalize the "Surfaces" table in `docs/plan.md`, then run
   `bob vault-sync run`.
5. **Bryan's checklist.** Put it in the final response; these steps are not automated.
   - **Triage now.** Release with Alt+N until NEXT ≤ 15 and PENDING ≤ 10; about 80
     statuses came from the old rules. If the 2026-10-01 pass already demoted tasks,
     re-promote keepers from the report's appendix or
     `/var/tmp/bob_task_status_hooks.log`.
   - **Check the dash in Obsidian:**
     - sections are exclusive;
     - TODAY updates at once when a link is added to or removed from today's daily note;
     - the chips are right.

     If TODAY doesn't refresh, report it; the fallback is one `bob-dashboard` block.

   - **MacBook and athena:**
     - reinstall `bob` if `hooks-resume` didn't;
     - rebuild and install Bob Mac Capture from master after `mac-lanes` CI is green;
     - run `bob plugins sync`.
   - **Parking swarm work:** `=x` close (→ `[/]`, carried), then Ctrl+Shift+Enter on the
     carried link with a one-line Work Log note. The task then waits in PENDING.
   - **Trial:** two weeks from `dash-lanes`. Keep the design if, on at least 10 of 14
     days, NEXT ≤ 15, PENDING ≤ 10, the morning review takes ≤ 5 minutes, and release
     gets used. If NEXT stays red, tighten the weekly prune first; age-based Next decay
     comes only after that, and never for Pending.

## Deliberately not doing

- **A `#today` tag or a file-path filter** (rejected; `#today` is a stopgap only).
- **Age-based Next decay:** only if the trial fails, and never for Pending.
- **A "worked, don't carry" `=x` outcome:** parking stays two steps until that proves
  annoying.
- **Blocked as a separate overlay** so lanes survive a block: recovery to Ready is
  accepted re-triage.
- **A capture spelling for commit or release:** Alt+N covers review.
- **Aligning hooks promotion with the Today definition:** the hooks still promote any
  live link under an open Pomodoro and transcluded dependencies. Also **cascading a
  release** to dependency-promoted Next tasks.
- **Renaming `[/]`, its In Progress type, or the `in_progress` picker group** (R12).
- **A stale report** (days since the last 🍅), the **single `bob-dashboard` fallback**
  (only if the Tasks refresh proves fragile; record a `PROPOSED FOLLOW-UP:`), and
  **automated triage** of the legacy statuses (Bryan's).

## Risks

- **An internal Tasks event.** `obsidian-tasks-plugin:reload-open-search-results` is
  internal to Tasks. A test pins the name; the fallback is cdx's single component.
- **Lane growth.** Without decay, lanes grow 15–20 a week. The caps, chips, lints, daily
  review and weekly prune exist for this, and the trial measures it.
- **Sticky dependency promotions** inflate NEXT; the weekly prune handles them. Record a
  follow-up if they dominate.
- **The MacBook is often offline.** Its steps are best-effort, and the checklist names
  what is left.
- **Interim demotion.** Keymaps demote until `link-toggle` deploys, and capture until
  `capture-toggle-lanes` is installed. The cutover note tells Bryan to delete link lines
  by hand in the meantime.
