---
tier: tale
title: "Tag open Pomodoro tasks from 2026-09-29 with #now"
goal:
  "Every not-done Obsidian task linked from a future or open Pomodoro in
  ~/bob/2026/20260929.md carries a whole-token #now, placed the way Alt+N places it, and
  the daily note stays unchanged."
size: medium
proposed_by: bbugyi200.apollo.3b
create_time: 2026-09-30 07:02:45
status: wip
---

# Plan: Tag open Pomodoro tasks from 2026-09-29 with #now

One agent can do this in a single pass. The set of tasks and the insertion rule are
fixed, and the only judgment left is a preflight abort if the vault drifted. That is a
tale of size medium: substantial, bounded, and implemented directly.

The `#now` token rule and the NOW query come from
`plan:202609/pomodoro_plan_budget_now_tag.md` (epic bob-cli-2o) and `docs/plan.md`.

## Decision

Tag all 73 tasks in the manifest below. Do not trim the set to the weekly cap of 15.

On the 2026-09-30 read, `bob plan` reported `NOW 0/15`. After this edit the same query
should report `NOW 73/15`, over the cap, with `now_cap_exceeded`. That overrun is the
intended result. `plan.strict` refuses a new over-cap theme. It does not refuse a `#now`
write. The meter turns red and the weekly review is where Bryan prunes. This tale only
adds the tag.

Inventory-label Pomodoros stay in the set. `SASE`, `MISC`, `NEW FEATURES`, and `LATER`
are storage names in `plan.inventory_labels`, and they are still open placeholder
entries on this note. `LATER` alone contributes 23 tasks. Excluding them would drop
tasks the request named.

## What to tag

Work in `~/bob`, edited in place. The ledger is `~/bob/2026/20260929.md`, from the
`## Pomodoros` heading through the next `## ` heading. Fenced code is skipped. There is
no fenced code in that section.

A Pomodoro entry is a column-0 checkbox item, `- [c] …`.

- **Open:** `c` is not `x`, `X`, or `-`. Completed and cancelled entries are history.
- **Future:** open, the body has an empty `()` placeholder, and it has no time range.
  This is the same rule as `next_future_pomodoro` in `src/native/capture_pomodoros.rs`,
  applied to every such entry rather than only the first.
- **Tag the union.** On the planning snapshot every open entry is also a future
  placeholder, so the two readings select the same 20 entries. If a timed open entry (a
  running Pomodoro) exists at implementation time, include it too. It is open.

An associated task is a block link on an indented line under that entry, up to the next
column-0 entry:

- Accept `[[target#^id]]`, `![[target#^id]]`, and `[[target#^id|alias]]`, with or
  without a 🍅 marker.
- Skip links inside `~~…~~`.
- The target is the note, with any `.md` suffix removed. An empty target means the daily
  note itself.
- Resolve the task by note path plus block id. A block id is not unique across the
  vault.

On the planning snapshot this produced 73 distinct links, and `bob plan` with
`BOB_DAY_FILE` pointed at that note reported the same 73 links. Every link resolved to
one not-done `#task` line. None of those lines already contained a whole-token `#now`.
Forty-eight are In Progress (`[/]`) and twenty-five are Next (`[*]`). None are Blocked
(`[?]`), Done, or Cancelled. None carry `#hide`. Two have a scheduled date on or before
2026-09-30, so they still match the NOW query: `sase_clean.md` `^work-all-beads` is
scheduled 2026-09-11, and `sase.md` `^bulk-gate-cmds` is scheduled 2026-08-22.

`^read-research` also exists on done tasks in `sase_remote.md` and `sase_art_links.md`.
The open GOALS link points only at `sase_agent_history.md#^read-research`. Do not tag
the other two.

## Manifest

Ledger order, as read on 2026-09-30. The nine notes are `sase.md` (58), `bob.md` (6),
`sase_memory.md` (3), and one each in `sase_goals.md`, `sase_agent_history.md`,
`sase_clean.md`, `sase_remote.md`, `sase_art_links.md`, and `sase_pager.md`.

| Pomodoro     | Note                    | Block                         | Status      |
| ------------ | ----------------------- | ----------------------------- | ----------- |
| BOB          | `bob.md`                | `^solo-id-capture`            | in progress |
| BOB          | `bob.md`                | `^better-capture-stop`        | in progress |
| GOALS        | `sase_goals.md`         | `^epic-roadmap`               | in progress |
| GOALS        | `bob.md`                | `^better-roadmaps`            | in progress |
| GOALS        | `sase_agent_history.md` | `^read-research`              | in progress |
| GOALS        | `sase.md`               | `^fix-telegram-leak`          | in progress |
| SASE         | `sase.md`               | `^recovery-panel`             | in progress |
| SASE         | `sase.md`               | `^final-ux`                   | in progress |
| SASE         | `sase.md`               | `^tui-cli`                    | next        |
| SASE         | `sase.md`               | `^node-finder`                | next        |
| SASE         | `sase.md`               | `^light-prompt-history`       | next        |
| SASE         | `sase.md`               | `^target-alias-providers`     | next        |
| SASE         | `sase.md`               | `^stash-trash`                | next        |
| SASE         | `sase.md`               | `^no-more-index-lock`         | next        |
| SASE         | `sase.md`               | `^fix-v-key`                  | in progress |
| DECKS        | `sase.md`               | `^card-blocks`                | in progress |
| DECKS        | `sase.md`               | `^p-key-for-decks`            | in progress |
| DECKS        | `sase.md`               | `^quote-keymap`               | in progress |
| DECKS        | `sase.md`               | `^fix-schema-mismatch`        | in progress |
| SASE         | `sase.md`               | `^bead-triage-job`            | in progress |
| SASE         | `sase.md`               | `^routine-groups`             | in progress |
| READ         | `sase.md`               | `^harden-services`            | in progress |
| READ         | `sase.md`               | `^clean-core`                 | in progress |
| READ         | `sase.md`               | `^output`                     | in progress |
| MISC         | `sase.md`               | `^fix-agent-tab-snap`         | next        |
| MISC         | `sase.md`               | `^web-piw-completion`         | next        |
| MISC         | `sase.md`               | `^bead-read`                  | next        |
| MISC         | `sase.md`               | `^multi-line-alternations`    | next        |
| MISC         | `bob.md`                | `^web-capture`                | next        |
| FINAL        | `sase.md`               | `^read-final-research`        | in progress |
| FINAL        | `bob.md`                | `^capture-project-files`      | next        |
| RENAME       | `sase.md`               | `^agent-renames`              | next        |
| CLEANUP      | `sase_clean.md`         | `^work-all-beads`             | in progress |
| REMOTE       | `sase_remote.md`        | `^use-sase-screenshot-to-fix` | in progress |
| SERVICE      | `sase.md`               | `^services`                   | in progress |
| NEW FEATURES | `sase.md`               | `^hold`                       | in progress |
| SUDO         | `sase.md`               | `^fix-claude-monitors`        | in progress |
| SUDO         | `sase.md`               | `^sudo`                       | in progress |
| FAST TESTS   | `sase.md`               | `^fast-tests`                 | in progress |
| QUEUE        | `sase.md`               | `^q-weight`                   | in progress |
| QUEUE        | `sase.md`               | `^q-weight-phases`            | next        |
| QUEUE        | `sase.md`               | `^q-weights-prj`              | next        |
| QUEUE        | `sase.md`               | `^q-weight-cap`               | next        |
| QUEUE        | `sase.md`               | `^q-weights-metadata`         | next        |
| SCHEDULE     | `sase.md`               | `^schedule`                   | next        |
| AUDIT MEMORY | `sase_memory.md`        | `^feature-flags`              | next        |
| AUDIT MEMORY | `sase_memory.md`        | `^review-sase-art`            | next        |
| AUDIT MEMORY | `sase_memory.md`        | `^memory-beads`               | next        |
| GATES        | `sase.md`               | `^bulk-gate-cmds`             | next        |
| GATES        | `sase.md`               | `^fast-tale-status`           | next        |
| LATER        | `sase.md`               | `^green-check-full`           | in progress |
| LATER        | `sase_art_links.md`     | `^artifact-event-store`       | next        |
| LATER        | `sase.md`               | `^stop-wait-renames`          | in progress |
| LATER        | `sase.md`               | `^improve-monitors`           | next        |
| LATER        | `sase.md`               | `^comma-big-h`                | in progress |
| LATER        | `sase.md`               | `^work-kills-waiting`         | in progress |
| LATER        | `sase.md`               | `^better-sbd-alias`           | in progress |
| LATER        | `sase_pager.md`         | `^fix-file-follow`            | in progress |
| LATER        | `sase.md`               | `^stale-epic-approved`        | in progress |
| LATER        | `sase.md`               | `^tui-screenshot`             | in progress |
| LATER        | `sase.md`               | `^tui-speedup-again`          | in progress |
| LATER        | `sase.md`               | `^just-fix-tui-screenshots`   | in progress |
| LATER        | `sase.md`               | `^bead-art-links`             | in progress |
| LATER        | `sase.md`               | `^critique-research-swarm`    | in progress |
| LATER        | `sase.md`               | `^dot-separators`             | in progress |
| LATER        | `sase.md`               | `^usage-service`              | in progress |
| LATER        | `sase.md`               | `^agent-clan-summaries`       | in progress |
| LATER        | `sase.md`               | `^fix-muse-waits`             | in progress |
| LATER        | `sase.md`               | `^xprompt-in-header`          | in progress |
| LATER        | `sase.md`               | `^agents-sub-tabs`            | in progress |
| LATER        | `sase.md`               | `^better-sidebar`             | in progress |
| LATER        | `bob.md`                | `^capture-stop`               | in progress |
| LATER        | `sase.md`               | `^release-v18`                | in progress |

## Insertion rule

Place the tag the way `addNowTagToLine` does in `bob-plugins`
`plugins/bob-navigation-hotkeys/main.js`. That is the Alt+N gesture. Copy that function,
plus `parseBulletPropertyFields`, `getTrailingBlockIdSpan`, and the two regexes below,
into a scratch script under `/tmp`. Do not modify bob-plugins. Re-find each task by its
trailing block id at implementation time. Planning-time line numbers go stale when
task-status grouping moves a line.

A whole-token `#now` is case-sensitive. It is preceded by the start of the line or
whitespace, and followed by the end of the line or whitespace. `#nowadays`, `#now/x`,
and `#NOW` do not count.

Regexes the function uses:

- Trailing block id: `[ \t]+\^([A-Za-z0-9-]+)[ \t]*$`
- Inline field: `\[([^\[\]\n]+?)::([^\]\n]*)\]`

Steps for one line:

1. If the line already has a whole-token `#now`, leave it unchanged.
2. The body ends at the trailing block id, or at the end of the line when there is no
   block id.
3. Trailing fields are the contiguous run of inline fields, with only whitespace between
   them, that ends at the body end. A field followed by prose is not in that run. The
   insertion point is the start of that run, or the body end when there is no such run.
4. Trim whitespace on both sides of the cut. Join `before`, ` #now`, and, when anything
   remains, a single space plus `after`.

The usual result on these tasks is the description, then ` #now`, then the existing
fields and block id:

```markdown
- [/] #task Read and act on [[core_schema_skew_outage_recovery_ux]]! #now
  [created::2026-09-25] ^recovery-panel
```

Three lines currently have two spaces before the first field. The join collapses that
gap to one space. Expect exactly these results:

```markdown
- [/] #task Allow `@@file#pomodoro` and `:id:` syntax for easier bulk capture! #now
  [created:: 2026-09-19] ^solo-id-capture
- [/] #task Add `w|weight` kwarg to `%q` to control how many slots in the queue the
  agent occupies! #now [created:: 2026-09-09] ^q-weight
- [*] #task Review and close all memory beads! #now [created:: 2026-09-06] [id::
  sase_memory__memory-beads] ^memory-beads
```

The field text stays byte-for-byte, including `[created:: 2026-09-19]` and
`[priority::high]`. Checkbox status, the `#task` token, the description, child bullets,
and work-log bullets stay as they are. `#now` is not a status and not a Next source.

Check the scratch script against these inputs before touching the vault. The expected
outputs are the plugin's own tests:

| Input                                              | Output                                                  |
| -------------------------------------------------- | ------------------------------------------------------- |
| `- [ ] #task Ship it`                              | `- [ ] #task Ship it #now`                              |
| `- [ ] #task Ship it ^a1`                          | `- [ ] #task Ship it #now ^a1`                          |
| `- [ ] #task Ship it [scheduled:: 2026-09-30] ^a1` | `- [ ] #task Ship it #now [scheduled:: 2026-09-30] ^a1` |
| `- [ ] #task Ship it [a:: 1] [b:: 2] ^a1`          | `- [ ] #task Ship it #now [a:: 1] [b:: 2] ^a1`          |
| `- [ ] #task Ship it #hide ^a1`                    | `- [ ] #task Ship it #hide #now ^a1`                    |
| `- [ ] #task Ship it #now ^a1`                     | `- [ ] #task Ship it #now ^a1`                          |
| `- [ ] #task Foo [a:: 1] bar [b:: 2] ^x`           | `- [ ] #task Foo [a:: 1] bar #now [b:: 2] ^x`           |

## Preflight, then write

Abort before any vault write when any check fails. Report the mismatch. Do not tag a
smaller set and do not invent replacements.

1. Re-read `~/bob/2026/20260929.md` and recompute the open-or-future link set with the
   rules above. The set of `(note, block id)` pairs must equal the manifest exactly.
2. In each note, find the one line whose trailing block id is that id. Zero matches or
   more than one match aborts.
3. That line is a `#task` whose checkbox is `[ ]`, `[*]`, or `[/]`. A missing `#task`,
   or a checkbox of `x`, `X`, `-`, or `?`, aborts.
4. Apply the insertion rule. The only byte changes on a line are the inserted ` #now`
   and the whitespace the join normalizes at the cut. Any other difference aborts.

Write the nine notes in place. Preserve each file's existing newline at end of file. Do
not reflow the rest of the note. A scratch script under `/tmp` is the right way to apply
73 replacements. Delete it when the writes succeed.

## Verify

- `git -C ~/bob diff --stat` lists only the nine notes above. `git -C ~/bob diff -U0`
  shows only the `#now` insertion, plus the three single-space collapses named above.
- Each manifest block id still has exactly one trailing-id line, and that line contains
  a whole-token `#now` before its first trailing inline field.
- `bob plan` reports `now.count` equal to 73 plus any `#now` tasks that already matched
  the NOW query before this edit. On the planning snapshot that base was 0, so the
  expected report is `NOW 73/15` and over. Today's ledger stays
  `PLAN 1/3 themes · 3/10 links` for `~/bob/2026/20260930.md`, because this tale does
  not edit that note.
- Run the NOW query with `bob query --format markdown --tasks-file` (or `--tasks`).
  Every manifest task appears. The query is:

```text
not done
tags include #now
is not blocked
tags do not include #hide
folder does not include _templates
path does not include _conflicts
(no scheduled date) OR (scheduled on or before today)
```

If a manifest task is absent from that result, report its path, block id, status, tags,
and scheduled date. Do not add further tags to force the count.

## Leave unchanged

- `~/bob/2026/20260929.md` and `~/bob/2026/20260930.md`. Today's open CLEANUP links
  (`^re-launch-failed`, `^clean-prompt-history`, `^decision-web`) and the GTD chore are
  not part of this set.
- Completed Pomodoros on the 29th, including struck links and the cancelled `^gtd` chore
  on that note.
- Child bullets, work logs, schedule logs, checkbox status, and task fields.
- `bob task-status-hooks`. It rewrites statuses and can move tasks between headings.
- The bob-cli checkout and bob-plugins. Do not commit or push `~/bob`. `bob-vault-sync`
  owns vault commits. Do not open the vault with `sase repo open`.
