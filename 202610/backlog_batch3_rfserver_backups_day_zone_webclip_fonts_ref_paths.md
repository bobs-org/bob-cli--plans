---
tier: epic
title: 'Backlog batch 3: rfserver suspension, backup tier freshness, Eastern day boundary,
  web-clip quote fidelity, and relative ref paths'
goal: Close five prioritized task beads by hardening rfserver's token suspension,
  proving every Athena backup tier advances after the rsync-24 fix, making bob's "today"
  follow Bryan's Eastern day on every host, stopping web-clip PDFs from splitting
  Highlights quote words, and making relative PDF paths map to their real reference
  note. The lander continues the backlog through further epics of at most five phases
  and five task beads until every project task bead is closed.
decisions:
  day_timezone:
    ask: Should bob decide "today" from a configured IANA timezone instead of the
      host's system zone?
    choices:
      config: bob honors a config timezone key set to America/New_York for every host
      host: Only apollo's system zone changes, via sudo; other UTC hosts keep the
        bug
    default: config
    why: Fixes every host and agent environment regardless of system zone, and is
      unit-testable
    answer: config
phases:
- id: rfserver-suspension
  title: Stop rfserver trusting X-Forwarded-For for token suspension
  size: small
  depends_on: []
  description: 'rfserver-suspension: resolve bob-cli-b3 by keying rfserver auth suspension
    on REMOTE_ADDR only, with a regression, a durable tailnet record, and a single-program
    restart.'
- id: backup-tier-freshness
  title: Prove daily, weekly, and monthly backup rotations advance
  size: medium
  depends_on: []
  description: 'backup-tier-freshness: resolve bob-cli-b2 by verifying the first post-fix
    daily and weekly rotations on Athena, diagnosing and fixing any tier that still
    fails, and recording monthly evidence.'
- id: day-timezone
  title: Make bob's today follow Bryan's Eastern day on every host
  size: medium
  depends_on: []
  description: 'day-timezone: resolve bob-cli-2q by applying the day_timezone decision
    so evening commands on apollo target the correct daily note and date stamps.'
- id: webclip-static-fonts
  title: Render web clips with static fonts so Highlights quotes keep whole words
  size: medium
  depends_on: []
  description: 'webclip-static-fonts: resolve bob-cli-85 by replacing the web-clip
    reader template''s variable woff2 faces with static faces so Chromium embeds no
    Type 3 body fonts.'
- id: ref-relative-paths
  title: Map relative PDF paths to their real reference notes
  size: small
  depends_on: []
  description: 'ref-relative-paths: resolve bob-cli-9a by absolutizing and normalizing
    PDF arguments before library mapping in bob ref sync and marker.'
proposed_by: bbugyi200.athena.bob-cli-an.6.land
proposed_with: claude/opus
parent_bead: bob-cli-an.6
decided_by: auto
create_time: 2026-10-23 17:27:14
status: done
bead_id: bob-cli-an.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md)
- **PARENT:** [202610/backlog_batch2_athena_hardening_and_ref_ids.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch2_athena_hardening_and_ref_ids.md)
- **BEAD:** [bob-cli-an.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.md)

# Backlog batch 3: rfserver suspension, backup tier freshness, Eastern day boundary, web-clip quote fidelity, and relative ref paths

## Scope and triage

This continues the backlog campaign of bounded epics. Batch 1 (`bob-cli-an`) and batch 2
(`bob-cli-an.6`, plan `202610/backlog_batch2_athena_hardening_and_ref_ids.md`) are
closed. Batch 2's lander closed `bob-cli-au`, `bob-cli-av`, `bob-cli-aw`, `bob-cli-1a`,
and `bob-cli-8o` with live evidence. It routed seven follow-ups (`bob-cli-b2` …
`bob-cli-b8`) and ran the obsolescence sweep that batch 1 deferred. The sweep closed 23
tasks as already resolved, each with evidence. It also recorded uncertain findings on
`bob-cli-3h`, `4o`, `73`, `91`, and `5m`, and snooze-prerequisite findings on
`bob-cli-3o` (met) and `6q` (not met). This epic owns exactly these five task beads; no
phase may add a sixth:

| Rank | Task         | Reason for priority                                                                                                                                                                        | Phase                   |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------- |
| 1    | `bob-cli-b3` | rfserver keys its token-failure suspension on a client-supplied `X-Forwarded-For`, so a local foothold can guess the `/sendmail` token without limit. The host was breached on 2026-10-17. | `rfserver-suspension`   |
| 2    | `bob-cli-b2` | Daily, weekly, and monthly `/home` recovery points are 6 days, 4 weeks, and 6 months old. The rsync-24 fix was proven only on hourly runs.                                                 | `backup-tier-freshness` |
| 3    | `bob-cli-2q` | Every evening after 20:00 EDT, bob on apollo (system zone `Etc/UTC`) reads and writes tomorrow's daily note and stamps the wrong dates (+1, oldest silent-corruption report).              | `day-timezone`          |
| 4    | `bob-cli-85` | Every web clip embeds Type 3 body fonts, so about 5.4% of Highlights quote words come back split (+2); the damage flows into ref notes and annotation tasks.                               | `webclip-static-fonts`  |
| 5    | `bob-cli-9a` | `bob ref sync` with a cwd-relative `lib/` path writes a duplicate top-level ref note and a duplicate reading task that sync to every host (+1).                                            | `ref-relative-paths`    |

Ranking: security first (`b3`), then content loss (`b2`), then silent corruption ordered
by reach and corroboration (`2q`, `85`, `9a`). These were deferred with reasons, not
silently absorbed:

- `bob-cli-7q` (consumption parity) is a design change to accepted decision
  `decisions:capture-link-drops-blocking-deps` ("plugin gestures keep PDG1
  consumption"), not a defect, so it needs its own reviewed decision.
- `bob-cli-4r` now affects 2 color-grouped notes; bob-cli-4w.8 fixed the 113-note shape.
- The deterministic CI blockers `bob-cli-b5` (bob-cli master lint red at
  `just check-scripts`), `bob-cli-b6` (`docs/develop/decisions.md` drift), and
  `bob-cli-b7` (bob-mac-capture RefsLibraryTests) rank after silent corruption. Each
  already raises its own TaskTriage gate, so Bryan can launch them directly.

On 2026-10-23, after the batch-2 landing and sweep, a fresh enumeration found **138
unresolved tasks: 134 ready and four snoozed** (`bob-cli-3o`, `bob-cli-6q`,
`bob-cli-aa`, `bob-cli-ah`). None were claimed or in progress. This is a snapshot, not a
substitute for the lander's fresh enumeration. Open epic plans at this snapshot were
`bob-cli-28`, `bob-cli-3j`, `bob-cli-5w`, `bob-cli-66`, `bob-cli-87`, `bob-cli-8b`,
`bob-cli-9k`, and `bob-cli-9k.8`. None owns this batch's tasks. Recheck ownership before
implementation.

## Working rules and handoffs

Read the assigned task, this epic's finalized decisions, and phase dependencies through
`sase bead read`. Reproduce against the current tree and host before changing anything.
A task that has meanwhile been fixed can be closed with independently checked evidence
instead of reimplemented. Record baseline, chosen fix, validation, and remaining
constraints on the task and phase. Use `sase bead close` with `done`, `superseded`, or a
human-authorized `canceled` resolution; never hand-edit the store or runtime-owned
statuses. Phase workers leave task beads open for the lander; the lander closes them
after rechecking evidence. Phase workers never create beads: record out-of-scope
discoveries as `PROPOSED FOLLOW-UP:` notes on your own phase.

Use `/sase_repo` before accessing any other repository (`tailnet` for Athena host
records, `chezmoi` for dotfiles and the shared bob config, `bob-plugins`,
`bob-mac-capture`). Read sidecar evidence only through `sase artifact read`. **Every
host phase must commit its tailnet record in the same turn as the host change, through
the turn's final declaration.** Batch 2's rfserver phase left no durable record until
its lander added tailnet `hosts/athena/rfserver/`.

Host phases gather nonprivileged evidence first and prepare exact files plus
install/rollback argv. Any root step goes through one reviewed `/sase_sudo` batch, and
plan approval does not remove that review. Never print credentials, tokens, or secret
config values; `/etc/environment` and `~/.config/supervisor/athena.conf` hold secrets,
so filter them out or use suppressed output. Preserve SSH, backups, monitoring, mail,
and service administration. A rejected gate is unfinished work, not a reason to call the
defect fixed. Use `/sase_monitor` for long commands or waits; never end a turn to wait.

bob-cli phases run the focused tests for the code they touch, then `just check` once on
their integrated change (use `/sase_monitor` if it runs long), and `just docs-check`
when they change docs. Follow the daily release train: do not edit package versions or
`CHANGELOG.md`, and do not publish a manual release. Never mutate the live vault during
validation except through a step the phase explicitly allows.

The five phases touch disjoint files and can run in parallel. `day-timezone`,
`webclip-static-fonts`, and `ref-relative-paths` all commit to bob-cli. Each rebases
through `sase stitch create`; resolve doc-index conflicts by keeping both sides.

## Phase rfserver-suspension — bob-cli-b3

Evidence (2026-10-23): `~/db/bin/main/rfuncs/path_dispatcher.py` `_get_client_address`
returns `environ["HTTP_X_FORWARDED_FOR"].split(",")[-1].strip()` whenever that header
exists, and falls back to `REMOTE_ADDR` only on `KeyError`. `authenticator.py` suspends
an address after more than 10 failed attempts in 10 minutes. A client that varies the
header is never suspended and can exhaust another address's bucket. No proxy fronts
rfserver; it binds `127.0.0.1:23401` only (bob-cli-aw).

1. Reproduce with a scratch server instance or a unit test, never against the live token
   or with repeated live bad-token requests. Each live bad-token probe counts toward the
   real suspension bucket, so use at most one, with a form body.
2. Make `_get_client_address` return `REMOTE_ADDR` only, and delete the header path. Add
   a regression beside `~/db/bin/tests/test_rfserver.py`: varying `X-Forwarded-For`
   values must share one suspension bucket and hit SUSPENDED after the limit.
3. `~/db/bin` is an uncommitted local checkout of `bbugyi200/scripts` that SASE cannot
   open. Edit it in place, then update tailnet `hosts/athena/rfserver/`: add the hunk to
   a patch file (or a second patch), copy the new regression into `tests/`, and update
   the README's verify, rollback, and residual-risk sections. Never record the token.
4. Restart only `tools:rfserver` through Bryan's user Supervisor. Verify that `ss -ltnH`
   shows `127.0.0.1:23401` only, that one loopback bad-token form POST returns 401, and
   that both rfserver regressions pass.

Acceptance: suspension buckets ignore client-supplied headers, regressions pass on the
live source, and the tailnet record matches the live files.

## Phase backup-tier-freshness — bob-cli-b2

Evidence (2026-10-23 16:43 EDT, `verify_root_cron.py` as bryan): hourly marker fresh;
daily marker 5.7 days old (2026-10-18), weekly 27.6 days (2026-09-26), monthly 183.6
days (2026-04-23). Batch 2 installed the rsync-24 commit fix in
`/usr/local/lib/athena-cron/backup.sh` at 15:06 EDT and proved two consecutive hourly
commits. Anacron spool: `cron.daily` last ran 2026-10-23 00:54, `cron.weekly`
2026-10-17, `cron.monthly` 2026-10-21. Daily and weekly are due on the first anacron
pass of 2026-10-24 (about 00:30–01:30 EDT), monthly about 2026-11-21. The tiers' own
failure cause was never observed: `/etc/anacrontab` mails job output to Gmail, not root
mail, and the `gog` Gmail token had expired.

1. If the current time is before 2026-10-24 03:00 EDT, hand off with `/sase_monitor`
   sleeping until then; do not run a live backup by hand to test scheduling.
2. Run `python3 -I hosts/athena/root-cron/verify_root_cron.py` (tailnet) as bryan and
   record each tier's marker age. Confirm `home/daily` and `home/weekly` are newer than
   their `-2` rotations, and compare one restore sample per advanced tier.
3. If a tier did not advance, find the failing step and exit code. Use a reviewed
   `/sase_sudo` read of the job output or root-readable logs, or the anacron mail if
   Bryan has restored `gog` auth (read-only; never re-authenticate yourself). Fix it in
   tailnet `hosts/athena/root-cron/` (for example a fatal rsync 23 on a specific source
   tree or a TOO_SOON interaction). Keep the timestamp, destination, symlink, TOO_SOON,
   and exit-23 guards. Extend `tests/test_boundary.py`, install through
   `install_root_cron.py` in one reviewed gate, and wait for the next scheduled run of
   that tier.
4. Monthly runs too late for this epic. If daily and weekly commit through the shared
   `backup.sh`, accept monthly on that evidence plus the scratch monthly exercise, and
   record its next due date. If monthly needs a tier-specific change, make it and record
   the residual check on `bob-cli-b2` for the lander.

Acceptance: daily and weekly `/home` rotations advance on scheduled runs after the fix,
with restore samples, or any remaining failure is diagnosed, fixed, and installed
through a reviewed gate. Tier freshness is recorded.

## Phase day-timezone — bob-cli-2q

Evidence: `src/native/env.rs` `current_datetime()` (around line 337) honors `BOB_NOW`
and `DATE`, then falls back to `Local::now().naive_local()` (around line 356).
`default_day_file()` derives `YYYY/YYYYMMDD.md` from it. apollo's `/etc/localtime` is
`Etc/UTC`, so from 20:00 EDT bob there targets tomorrow. The one-time freshness seed on
2026-09-30 stamped 390 tasks with the UTC date (+1 evidence on the bead). `chrono` is
the only date crate today. Bryan's bob config is chezmoi-managed
(`home/dot_config/bob/config.yml`).

> [!decision] day_timezone = config Add an optional `timezone:` key (IANA name) to bob's
> config. When it is set, `current_datetime()` and every "today"/date-stamp path
> (including `bob plan`, `bob capture`, task-status hooks, freshness, pomodoro, and
> notify) compute local time in that zone. `BOB_NOW`/`DATE` overrides keep precedence,
> and an unset key keeps the system zone. Add a timezone database dependency (for
> example `chrono-tz` or `jiff`) only if needed, and check its package-size cost against
> the cap that `bob-cli-ai` tracks. Reject an invalid zone with a clear config error.
> Set `timezone: America/New_York` in the chezmoi-managed config, and apply it on apollo
> and athena. Document the key in `docs/reference/configuration.md`, and add tests that
> pin a UTC system clock across 20:00–24:00 EDT.

> [!decision] day_timezone = host Leave bob unchanged. Set apollo's system timezone to
> `America/New_York` through a reviewed `/sase_sudo` gate (`timedatectl set-timezone`).
> Record the change and its rollback in tailnet under `hosts/apollo/`, and check that
> apollo's cron and vault-sync timers still behave. Document in `docs/vault-git-sync.md`
> that bob hosts must run in Bryan's Eastern zone.

Verify on apollo, without waiting for 20:00 EDT, by running the relevant `bob plan` and
`bob capture --dry-run` paths with a pinned UTC instant after 00:00 UTC. They must name
the Eastern day's note. Do not repair the historical 2026-09-30 freshness stamps; record
them as residual data on the task.

Acceptance: on every host, bob's "today" and date stamps follow Bryan's Eastern day
between 20:00 and 24:00 EDT, proven by tests or the host change plus an apollo check.

## Phase webclip-static-fonts — bob-cli-85

Evidence: `scripts/web_clip/template/reader.css` declares `@font-face` rules for
`*-wght-*.woff2` variable faces (`format("woff2-variations")`), which `src/scripts.rs`
embeds and installs under `web_clip/fonts/`. Chromium/Skia embeds variable fonts as Type
3, and PDFKit then splits words: 88 of 1631 quote words (5.4%) were damaged across 5 web
clips. Static TrueType faces embed as CID TrueType (1.1%, line-join merges only), and
pandoc+xelatex measured 0.19%. bob-cli-87.1 already vendored static faces in
`assets/print_fonts/` (Source Serif 4 Text, Inter, JetBrains Mono, Noto Emoji, plus
`OFL.txt` and a README with SHA-256s) for note snapshots.

1. Reproduce: render a fixture article through the web-clip route and run `pdffonts` on
   the PDF. Body faces should show as Type 3.
2. Switch `reader.css` and the installed font set to the static faces, reusing the bytes
   bob-cli-87.1 already embeds rather than embedding a second copy. Keep weights,
   italics, and the latin-ext coverage the variable faces provided. Remove the
   now-unused woff2 files from `src/scripts.rs` and the installer, and update
   `scripts/web_clip/fonts/README.md` and the license notes.
3. Add a regression that fails when any body font in a rendered web clip is Type 3. Run
   the web-clip adapter self-test. Compare `pdftotext` word fidelity before and after on
   the fixture. If `ssh mac` is reachable, rerun the PDFKit probe on one fresh clip; the
   research asset is cited in the bead's +1. Do not re-render existing library PDFs.
4. Measure package size against `bob-cli-ai`'s cap and record it. If static faces still
   embed as Type 3, or the PDFKit damage stays above about 1.5%, stop and record the
   evidence plus the xelatex re-typeset alternative as a `PROPOSED FOLLOW-UP`, rather
   than switching rendering engines in this phase.

Acceptance: new web clips embed no Type 3 body fonts, a regression enforces that, and
measured quote-word damage drops to the CID TrueType level.

## Phase ref-relative-paths — bob-cli-9a

Evidence: `src/native/highlights_ref/note.rs` `required_path` (around line 703) only
expands `~`. `io.rs` `pdf_path_metadata` then fails `strip_prefix(&config.lib_dir)` for
a cwd-relative path and falls through to the legacy flat `ref/<stem>.md` target with no
`ref_type`. From `~/bob`, `bob ref sync lib/chat/<stem>.pdf --dry-run` plans
`note_action: create` at `ref/<stem>.md` plus a `-2` reading task. The absolute path
correctly reports `ref/chat/<stem>.md`, `note_action: none`.

1. Add failing CLI regressions that run `sync` and `marker` with a relative
   `lib/<type>/` path from a scratch vault root, plus a `./`/`..` variant.
2. Make the PDF argument absolute against the current directory and lexically normalize
   it before any library mapping. Do this in the shared path helper, so every `bob ref`
   subcommand taking a PDF path (`sync`, `marker`, and any other `required_path` caller)
   benefits. Do not resolve symlinks unless the existing absolute path behavior already
   does.
3. Update `docs/ref.md` if it implies absolute paths. Run the highlights_ref tests and
   `just check`.

Acceptance: relative, `./`, and absolute spellings of the same library PDF plan the same
note and reading task, and no command creates a flat `ref/<stem>.md` for an in-library
PDF.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a sixth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and host state, including whether
another epic changed a shared service, cron, backup, config, or reference path. Confirm
each host phase's tailnet record matches the live files. Run appropriate integrated
checks once and document genuine preexisting failures with their existing task owners.
Require real evidence before closing each task: a host change with live verification,
tier freshness with restore samples, timezone tests or a host check, a pdffonts and
fidelity measurement, and relative-path regressions.

Because `sase plan propose` terminates the runner without running the commit finalizer,
the lander must commit every repository it changed (bob-cli, tailnet, chezmoi, the plans
sidecar status edit, and any other opened repo) with `/sase_git_commit`, which this
approved plan explicitly authorizes for that purpose, before proposing the successor.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include `bob-cli-8a`, `bob-cli-al`, `bob-cli-4r`, `bob-cli-79`, and
   `bob-cli-b8` for identity/write defects; deterministic CI blockers `bob-cli-b5`,
   `bob-cli-b6`, and `bob-cli-b7` unless already fixed through their triage gates;
   heavily corroborated `bob-cli-7i`, `bob-cli-6v`, `bob-cli-3w`, and `bob-cli-5l`
   follow closely; `bob-cli-b4`, `bob-cli-8r`, and `bob-cli-9x` cover Athena usability
   and detection. This is guidance, not a preassigned batch, and concurrent work may
   obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes. Verify real prerequisites, not just wake timestamps. Work
   other actionable tasks first. If only gated tasks remain, the next bounded epic must
   own the readiness check and supported `/sase_monitor` or `/sase_questions` handoff
   needed to resume; do not abandon the chain, spin on fresh identical epics, or close a
   valid task merely to reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.
