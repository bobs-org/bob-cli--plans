---
tier: tale
title: Merge BOB, BLOG, PYPI, and HACK into the active Pomodoro themes
goal:
  Retire four project-style themes by merging their ledger history into BUILD, WRITE,
  READ, and HOME, pushed and verified.
size: small
decisions:
  blog_split:
    ask:
      Should BLOG's 17 article-reading sessions be retagged READ before the rest of BLOG
      merges into WRITE?
    choices:
      split:
        Retag 17 article-reading entries (Oct 18–23) to READ; merge the other 25 into
        WRITE
      whole:
        Merge all 42 BLOG entries into WRITE; about 4h50m of article reading counts as
        WRITE
    default: split
    why:
      Since Oct 23 this ref/blogs reading is logged as READ; it is 41% of BLOG's logged
      minutes
    answer: split
proposed_by: bbugyi200.apollo.ai
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 18:52:18
status: wip
---

# Plan: Merge BOB, BLOG, PYPI, and HACK into the active activity themes

## Goal

Retire the four project-style Pomodoro themes that still show in Recent (BOB, BLOG,
PYPI, HACK) by merging their ledger history into the activity themes Bryan has used
since 2026-10-23 (BUILD, FIX, HOME, WRITE, READ, plus exempt GTD). This follows the VIBE
→ BUILD rename already made on the Mac (`bob themes rename: VIBE → BUILD`, vault commit
`d3d0c9a8`).

This is a **vault data migration only**. There are no bob-cli source, docs, memory, or
bob-plugins changes. The deliverable is a handful of vault commits, pushed through
vault-sync, plus a verification report.

## Mapping and evidence

Survey of every canonical daily note (`YYYY/YYYYMMDD.md`, `## Pomodoros` entries) on
2026-10-26:

| Old  | Entries / days | New                          | Why                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---- | -------------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BOB  | 72 / 26        | BUILD                        | 1725 timed min; 91% are bob feature tasks (`bob`, `bob_gtd`, `bob_git` notes). Since Oct 23, bob work (`bob#^root-parent`, `bob#^theme-day-strip-bar`, …) is logged under BUILD. The ~145 min of `fix-*` tasks are too small to split. Includes composites `BOB + STARTUP` and `GTD + BOB`.                                                                                                                                  |
| HACK | 6 / 6          | HOME                         | Every HACK entry carries the Plex-compromise chores (`dev_plex_hack_patch#^sell-stocks-now`, `^external-backup`, `dev#^ref-cp1500pfclcd-battery-swap-and-athena`). On Oct 23 the identical link list moved from the HACK placeholder to a HOME placeholder.                                                                                                                                                                  |
| PYPI | 10 / 6         | READ                         | Closed sessions are mostly reading `sase_pypi_releases` reference notes (`ref-agentic-coding-is-straining-ci-heres` ×3, `ref-pypi-release-reliability-…`). On Oct 23 the next READ sessions continued those same refs. Only one 25-min task session (`handle-manual-pypi`) is not reading, too small to split.                                                                                                               |
| BLOG | 42 / 14        | WRITE (+ READ, see decision) | Oct 7–17 is writing the SASE launch post (`write-introduction`, the `ref/chat/sase_launch_post_*` outline reports). That work is WRITE, matching how `ref/chat/sase_blog_writers_block_task_plan` is logged under WRITE now. Oct 18–23 is mostly reading external articles in `ref/blogs/` (Gas Town manual, "Redesigning source control at Anthropic", "How the Claude Code team does code review"), which READ now covers. |

`bob themes rename OLD NEW --merge -d` passes for all four pairs with no refusals.
`plan.exempt` is `[GTD]` and `plan.inventory_labels` is `[LATER, MISC, NEW FEATURES]`,
there are no Stats theme filters, and no journal state (hide or pin) exists on any
source or target key. Dry-run footprint: BOB 26 files / 72 entries, BLOG 14 / 42, PYPI 6
/ 10, HACK 6 / 6. None of the four themes is in today's note, `_templates/daily.md`, or
any future-dated note. Only past notes (2026-08-27 … 2026-10-24) change.

## Approach

Use the supported `bob themes rename OLD NEW --merge` for every whole-theme merge (PT10
in `docs/pomodoro-themes.md`). It takes `bob_sync.lock`, pre-syncs, rewrites only the
theme component in ` — NAME` tails byte-for-byte, appends a `theme.rename` journal event
(so `renamed_reappeared` flags any later muscle-memory reuse), verifies by rescanning,
makes one commit `bob themes rename: OLD → NEW`, and post-syncs. Undo is `git revert`.

Run it on the host executing this plan (apollo or athena; both hold the synced vault at
`~/bob`). Theme history is vault data, so the Mac sees the result after its next
`com.bbugyi.bob-vault-sync` cycle. Running from the Mac is not required. The journal
event lands in that host's `.bob/themes/<host>.jsonl`, and the fold reads every host's
file.

Rejected alternatives:

- **Hide instead of merge** (`bob themes hide`). Hiding only affects Recent, so Stats
  would stay split across retired names. Bryan asked for a merge.
- **Hand `sed` over daily notes for whole themes.** This skips the preview, stale-note
  check, verification, journal event, and single-commit undo.
- **Per-entry reclassification of all four themes.** BOB, PYPI, and HACK are ≥80% one
  activity, so splitting them adds hand edits for little accuracy. Only BLOG is
  genuinely mixed (decision `blog_split`).
- **Adding a scoped or per-entry rename to bob-cli.** That is a new contract (PT10
  verify requires OLD to be absent) for a one-off. A 17-line, lock-guarded edit is
  enough.

## Steps

Use a scratch directory outside the vault and outside any repo checkout (`mktemp -d`)
for snapshots, reports, and the edit script. Run Python with `python3 -I`.

### 1. Preflight

1. Run `bob vault-sync`, then `bob vault-sync status --json`. Require
   `last_error: null`, `conflicts: []`, and `local_sha == remote_sha`. Then run
   `git -C ~/bob status --porcelain`. If paths are dirty, run `bob vault-sync` once
   more. If they stay dirty, stop and report.
2. Snapshot `bob themes list -a -f json` to `before.json`. Record each theme's `entries`
   / `days_used` and the sum of `entries` over all themes (653 at planning time) and
   `counts.known` (81).
3. Dry-run all four renames with `-d -f json --merge` and confirm `ok: true` and the
   footprint above (BOB 26 / 72, BLOG 14 / 42, PYPI 6 / 10, HACK 6 / 6). If a count
   differs, find out why before writing anything. A new entry Bryan added since planning
   is fine; an unexplained change means stop and report.

### 2. BLOG reading split → READ

> [!decision] blog_split = split Do this step. Retag exactly the 17 BLOG entries below
> to READ before step 3.

> [!decision] blog_split = whole Skip this step entirely. BLOG merges into WRITE as a
> whole in step 3.

**Rule:** a BLOG entry moves to READ if and only if every Task Link under it points at a
`ref/blogs/` reference task. Mixed entries, `ref/chat` and `ref/notes` entries, and
link-less entries stay BLOG and merge into WRITE. First recompute the set with this rule
from the vault: resolve each child `[[…#^block]]` to the `#ref [[ref/<kind>/…]]` task
line that owns `^block`. Assert that the result equals this list exactly. Line numbers
are as of planning and are only hints. Match on the exact full line text, which is
unique in each file's `## Pomodoros` section. All lines are LF-terminated with no
trailing spaces.

| File               | Line | Exact current line                       | Read                       |
| ------------------ | ---- | ---------------------------------------- | -------------------------- |
| `2026/20261018.md` | 42   | `- [x] (**1350-1405** [t:: 15m]) — BLOG` | Gas Town                   |
| `2026/20261018.md` | 52   | `- [x] (**1615-1625** [t:: 10m]) — BLOG` | Gas Town                   |
| `2026/20261019.md` | 57   | `- [x] (**1440-1505** [t:: 25m]) — BLOG` | Gas Town                   |
| `2026/20261020.md` | 38   | `- [x] (**1025-1050** [t:: 25m]) — BLOG` | Redesigning source control |
| `2026/20261020.md` | 51   | `- [x] (**1530-1545** [t:: 15m]) — BLOG` | Redesigning source control |
| `2026/20261020.md` | 57   | `- [x] (**1700-1725** [t:: 25m]) — BLOG` | Redesigning source control |
| `2026/20261020.md` | 59   | `- [x] (**1800-1810** [t:: 10m]) — BLOG` | Redesigning source control |
| `2026/20261020.md` | 61   | `- [ ] (**1820-1840** [t:: 20m]) — BLOG` | Redesigning source control |
| `2026/20261021.md` | 33   | `- [x] (**0820-0830** [t:: 10m]) — BLOG` | Redesigning source control |
| `2026/20261021.md` | 35   | `- [x] (**0855-0910** [t:: 15m]) — BLOG` | Redesigning source control |
| `2026/20261021.md` | 37   | `- [x] (**1045-1110** [t:: 25m]) — BLOG` | Redesigning source control |
| `2026/20261021.md` | 39   | `- [x] (**1800-1825** [t:: 25m]) — BLOG` | Redesigning source control |
| `2026/20261021.md` | 41   | `- [x] (**1900-1910** [t:: 10m]) — BLOG` | Redesigning source control |
| `2026/20261022.md` | 29   | `- [x] (**0935-1000** [t:: 25m]) — BLOG` | Solving code review        |
| `2026/20261022.md` | 31   | `- [x] (**1235-1255** [t:: 20m]) — BLOG` | Solving code review        |
| `2026/20261023.md` | 33   | `- [x] (**0740-0805** [t:: 25m]) — BLOG` | Solving code review        |
| `2026/20261023.md` | 35   | `- [x] (**0855-0905** [t:: 10m]) — BLOG` | Solving code review        |

Mixed BLOG entries that intentionally stay (→ WRITE) include `20261017` 1750, `20261018`
1745, `20261019` 1825, `20261020` 1310 and 1620, `20261021`'s placeholder, and
`20261022` 1315.

**Edit procedure.** apollo's `bob-vault-sync.service` is an inotify watcher that runs
`bob vault-sync -q` within ~20 s of any vault change. Make the edit while holding the
same lock so no sync sees a half-edited set:

1. Write an edit script in the scratch dir. Under the lock it must:
   - require `git -C ~/bob status --porcelain` to be empty, else exit nonzero with no
     writes;
   - read all six files and assert every expected line is present exactly once inside
     `## Pomodoros`, else exit nonzero with no writes;
   - replace only the trailing ` — BLOG` with ` — READ` on those 17 lines, keeping every
     other byte;
   - write each file atomically (temp file in the same directory, then `os.replace`),
     restoring every before-image if any write fails.
2. Run it as `flock -w 60 "${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock" python3 -I <script>`.
   This is the lock bob uses (flock(2); `/run/user/1000/bob_sync.lock` on apollo, the
   same path the user service resolves).
3. After the lock is released, run `bob vault-sync`. The watcher may commit first, which
   is fine. Commit only through vault-sync, never with a manual `git commit` in the
   vault. Find the commit with `git -C ~/bob log -1 --format=%H -- 2026/20261023.md`.
   Then check that `git -C ~/bob show --stat` lists exactly those six files and that
   `git -C ~/bob show -U0 <sha>` shows 17 `— BLOG` → `— READ` line pairs and nothing
   else.

### 3. Whole-theme merges

Run each one live with JSON output and save every report:

```bash
bob themes rename HACK HOME  --merge -f json
bob themes rename PYPI READ  --merge -f json
bob themes rename BOB  BUILD --merge -f json
bob themes rename BLOG WRITE --merge -f json
```

- Each must exit 0 with `ok: true`, a `commit.sha`, and `sync.post` not failed. Expected
  `entries`: HACK 6, PYPI 10, BOB 72, and BLOG 25 under `split` or 42 under `whole`.
- `lock_busy`: the watcher is mid-sync. Wait ~10 s and retry, up to 3 times.
  `stale_note`: re-run once. `verify_failed`, `sync_failed`, or any other refusal: stop
  (bob restores the before-images itself) and report the JSON verbatim.
- BLOG → WRITE runs last so that under `split` it only sees the remaining 25 entries.

### 4. Verify

1. `bob themes list -a -f json`:
   - keys `bob`, `blog`, `pypi`, `hack` are absent;
   - `renamed_reappeared` is `[]` and `hints` is `[]`;
   - `counts.known` is 4 lower than in `before.json`;
   - the sum of `entries` over all themes equals the `before.json` sum, plus only
     entries Bryan added today in the meantime. No tail held two colliding components,
     so nothing collapses.
2. Per-target deltas against `before.json` (minus any of today's live additions): BUILD
   +72, HOME +6, READ +27 / WRITE +25 under `split`, or READ +10 / WRITE +42 under
   `whole`. Expected absolute values at planning time:
   - BUILD 91 entries / 29 days and HOME 10 / 10;
   - under `split`, READ 45 / 15 and WRITE 38 / 17;
   - under `whole`, READ 28 / 15 and WRITE 55 / 17.
3. Independent sanity scan (read-only Python over `~/bob/20*/YYYYMMDD.md`,
   `## Pomodoros`, fenced lines skipped): no ` — NAME` tail contains a `+`-component
   that normalizes to `bob`, `blog`, `pypi`, or `hack`.
4. `bob themes list` (human) shows only GTD, BUILD, FIX, HOME, WRITE, READ, with no hint
   lines.
5. `bob vault-sync status --json`: `last_error: null`, `conflicts: []`,
   `local_sha == remote_sha`. `git -C ~/bob log --oneline -8` shows the four
   `bob themes rename:` commits, plus the split's vault-sync commit under `split`.
6. Best-effort Mac check (it is offline when the lid is closed):
   `ssh -o ConnectTimeout=10 mac 'zsh -lc "bob vault-sync status --json; bob themes list"'`.
   If it is unreachable, or has not synced yet, say so. The LaunchAgent picks up the
   commits on its next cycle, and the catalog cache rebuilds from the changed files'
   fingerprints.

### 5. Report back

Give Bryan:

- the mapping applied and per-theme entry counts;
- every commit SHA, in order;
- the final `bob themes list` output and the Mac check result;
- the undo runbook: `git -C ~/bob revert <sha>` for each commit in reverse order (BLOG →
  WRITE first, the split commit last), then `bob vault-sync`. Reverting a rename commit
  also removes its journal line, because the event is part of that commit.

## Out of scope

- The 71 already-hidden older themes, the finished VIBE → BUILD rename, and hue pins.
  After the merge, BUILD yellow, FIX blue, HOME violet, WRITE magenta, and READ green
  are all distinct, so no collision hint is expected.
- Any bob-cli, bob-plugins, Bob Mac Capture, docs, or memory change. PT10 in
  `docs/pomodoro-themes.md` already describes this flow. Docs and glossary examples that
  mention BOB or BLOG are illustrative and stay true.
- Prose, task text, and `.bob/ref-sources/` mentions of these names. They are not ledger
  tails, and rename never touches them.
