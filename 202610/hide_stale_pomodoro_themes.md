---
tier: tale
title: Hide Pomodoro themes unused in the last 7 days
goal:
  Hide every Pomodoro theme last used before the cutoff so Recent menus on every host
  show only recent themes.
size: small
decisions:
  cutoff:
    ask: Which days count as "the last 7 days" when picking themes to hide?
    choices:
      week:
        Keep themes used Oct 20-26; also hides MEMORY, BUGS, PUBLIC (last used Oct 19);
        71 hidden
      eight:
        Keep themes used since Oct 19, so MEMORY, BUGS, PUBLIC stay visible; 68 hidden
    default: week
    why:
      Matches bob's own N-day windows (90-day list, 30-day hints), which count today as
      day one
    answer: week
proposed_by: bbugyi200.apollo.ah
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 18:10:04
status: wip
---

# Hide Pomodoro themes not used in the last 7 days

## Goal

Hide every Pomodoro theme whose last use was before the cutoff, so that only recently
used themes appear in the Recent rows of the Pomodoro-name menus on the MacBook (Bob Mac
Capture), in Obsidian, and in capture completion. This is a one-time data operation run
through the existing `bob themes hide` writer. **No bob-cli source, docs, tests, or
memory change**, and nothing is committed in the bob-cli repo.

## What the reviewer should know first

- **Not host-specific.** Theme preferences live in `.bob/themes/<host>.jsonl`. Each host
  appends only to its own file, but every reader folds **all** hosts' files
  (`docs/pomodoro-themes.md` PT4; decision `theme-prefs-live-in-per-host-journal`). A
  hide written on apollo reaches the MacBook as soon as vault Git sync runs on both
  machines. So this runs on apollo, which is always online. The MacBook is only up while
  its lid is open, so it is checked best-effort at the end.
- **"Hide" is the only off switch, and it only affects suggestions.** Hiding removes a
  theme from **Recent** rows and from the default `bob themes list`. It does not touch
  today's ledger rows, typing the name, history, Stats, or colors (decision
  `theme-hide-is-recent-only`). Nothing is deleted.
- **Hides stick even after the theme is used again.** Using a hidden name never unhides
  it. If you start a `JOB` Pomodoro tomorrow, `JOB` still won't show up in Recent until
  you run `bob themes unhide JOB` (or press Ctrl+D on its ghost row in a Pomodoro-name
  menu). The list includes long-running themes: JOB (48 entries), SASE (33), MEMORY
  (19), SHIT (16), GOALS (14). The final report prints a ready-to-paste undo command.
- **The cutoff is a fixed date, not "7 days before whenever this runs".** It is fixed to
  the request date, 2026-10-26, so the list you review is the list that gets hidden. If
  this runs later, any theme used after the snapshot has a newer `last_used`, stays
  above the cutoff, and is kept automatically.

> [!decision] cutoff = week Hide every visible theme with `last_used < 2026-10-20`.

> [!decision] cutoff = eight Hide every visible theme with `last_used < 2026-10-19`.

## Snapshot (catalog as of 2026-10-26 on apollo)

81 known themes, 0 hidden, 5 in today's ledger, no fold warnings.

**Kept under either choice (10):** GTD (exempt, today), BUILD, FIX, HOME, WRITE, READ,
BOB, BLOG, PYPI, HACK.

**Kept only under `eight` (3):** MEMORY, BUGS, PUBLIC (all last used Oct 19).

**Hidden under either choice (68):** SASE, ATHENA, JOB, DEV, GOALS, LOVE, SHIT, SASE
V18, BEADS, FIRE, FIX-APOLLO, HERMES, JITUX, RESTART MAC, SASE-V18, RESEARCH, SASE.MD,
SASE V19, LISTEN, PAGER, CLEANUP, BETS, REMOTE, TOOL, AUDIT MEMORY, DECKS, FAST TESTS,
FINAL, LATER, MISC, NEW FEATURES, QUEUE, RENAME, SERVICE, SUDO, GATES, SCHEDULE,
CAPTURE, RELEASE, RESTARTS, RELAUNCH, SUBTABS, TOOLS, PROJECTS, PACKET, ARTIFACTS, MUSE,
SCREENSHOTS, SCREENSHOT, SASE AGENTS, ALIAS DEFS, PRINT COMMAND, USAGE, MESSAGES,
VERIFY, FIX KILL AND EDIT, MEMORY BEADS, SWAP POMS, NEW MODELS, FIXES, SETUP, MEMORY
WEBS, LINKS AND SETUP, EMAIL, MAC AND APOLLO, PLAN MEMORY, AFTER TUI FIX, STARTUP.

Many names contain spaces (`SASE V18`, `RESTART MAC`, `FIX KILL AND EDIT`, ...). Each
one must reach `bob` as a single argv element.

## Steps

Run everything on apollo against the vault at `~/bob`, using the installed `bob`
(`~/.cargo/bin/bob`, which already ships `bob themes hide`). Keep scratch files in a
fresh `mktemp -d` directory **outside** the vault and outside any repo. Run Python with
`-I` if you use it.

1. **Preflight (read-only).**
   - `bob --version`, and `bob themes hide --help` to confirm the writer exists.
   - `bob themes list -a -F -f json > "$tmp/before.json"`. `-F` rebuilds from the daily
     notes, ignoring the cache. Confirm `warnings` is empty and record `today`,
     `counts.known`, and `counts.hidden`.
   - `git -C ~/bob status --short .bob/themes`. Note any pre-existing dirt, and don't
     touch it.

2. **Compute the candidate list.** From `before.json`, select themes with
   `hidden == false` and `last_used < CUTOFF`, where CUTOFF is `2026-10-20` (`week`) or
   `2026-10-19` (`eight`). Write one name per line to `$tmp/hide.txt`, using the
   catalog's display `name` exactly as reported.
   - Diff the result against the snapshot above. Some differences are expected: themes
     used since the snapshot drop out, and themes already hidden are skipped. A theme
     absent from the snapshot that shows up as a candidate is unexpected. If that
     happens, or if the candidate list grows by more than a handful versus the snapshot,
     stop and report instead of hiding.
   - Never select a theme with `today` set, or with `last_used >= CUTOFF`.

3. **Hide them in one writer call.** Load the names into a bash array with
   `mapfile -t names < "$tmp/hide.txt"`, then run
   `bob themes hide -f json -- "${names[@]}" | tee "$tmp/hide.json"`.
   - One call takes `bob_sync.lock` once and appends one `theme.hide` event per name,
     all with the same timestamp. Already-hidden names are skipped.
   - On `lock_busy` (vault-sync holds the lock while it runs), wait about 15 s and
     retry, at most 5 times. On any other refusal (`unknown_theme`, `io`), stop and
     report the JSON. Don't retry with edited names.
   - Never hand-edit `.bob/themes/*.jsonl`; bob is the only writer.

4. **Verify the fold.**
   - `bob themes list -H -f json`: the hidden set equals the old hidden set plus every
     name in `hide.txt`.
   - `bob themes list -f json` (default 90-day visible list): it contains exactly the
     kept themes. Under `week`, as of the snapshot, those are GTD, BUILD, FIX, HOME,
     WRITE, READ, BOB, BLOG, PYPI, HACK, plus anything used since.
   - `bob themes list -a -f json`: `counts.known` is unchanged, so nothing was removed
     from the catalog.
   - The tail of `~/bob/.bob/themes/apollo.jsonl` holds one `theme.hide` line per hidden
     name, with `"host":"apollo"` and `"via":"bob-cli"`. If the events went to a
     different `<host>.jsonl`, report that. It is still harmless, because fold reads
     every host file.

5. **Confirm it synced.** `bob-vault-sync.service` normally commits the journal on its
   own as `vault(apollo): 1 file - .bob/themes/apollo.jsonl`.
   - Poll `git -C ~/bob log -1 --format='%h %s' -- .bob/themes/apollo.jsonl` and
     `git -C ~/bob status --short .bob/themes` for up to about 3 minutes.
   - If the change is still uncommitted after that, run `bob vault-sync` once.
   - Finish with `bob vault-sync status --json` and record the result.

6. **Check the MacBook (best-effort, read-only).** Run
   `ssh -o ConnectTimeout=10 mac 'zsh -lc "bob themes list -H -f json"'`.
   - If the Mac is offline, or hasn't pulled yet (its LaunchAgent
     `com.bbugyi.bob-vault-sync` syncs on its own schedule), say so. That is expected,
     not a failure.
   - Don't run any writer or sync on the Mac.

## Final report

- The cutoff used and the number hidden.
- The hidden names, the kept names, and any differences from the snapshot.
- The sync commit hash, and the MacBook check result.
- A ready-to-paste undo command that restores every hidden theme in one call, with each
  name quoted, e.g. `bob themes unhide "SASE V18" JOB SASE …`.
- A reminder: a hidden theme stays out of Recent even after it is used again, until it
  is unhidden by command or with Ctrl+D on its ghost row.

## Out of scope

- A reusable bulk filter such as `bob themes hide --unused-for 7` or an automatic
  stale-theme hide. If the user wants this to recur, file a feature bead afterwards
  rather than adding CLI surface here.
- Any change to Stats, colors, renames, or the hue rule.
