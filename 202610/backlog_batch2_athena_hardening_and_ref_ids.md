---
tier: epic
title:
  "Backlog batch 2: Athena root, listener, and backup hardening plus reference ID
  allocation"
goal:
  Close five prioritized task beads by removing the remaining Bryan-to-root cron chains,
  committing hourly-tier backup rotations despite benign rsync churn, restricting
  Athena's remaining wildcard control listeners, and making new reference IDs follow
  their stems without cross-type collisions. The lander continues the backlog through
  further epics of at most five phases and five task beads until every project task bead
  is closed.
decisions:
  multivisor_web_access:
    ask:
      Should the Multivisor web UI on port 8100 stay reachable from Bryan's other
      tailnet devices?
    choices:
      tailnet:
        Allow 8100 from loopback and tailnet only; the UI still opens on other tailnet
        devices
      loopback: Bind 8100 to 127.0.0.1 only; remote use needs an SSH tunnel
    default: tailnet
    why: Drops LAN exposure without breaking remote use of the password-protected UI
    answer: tailnet
phases:
  - id: periodic-cron
    title: Move daily, weekly, and monthly root cron onto root-owned code
    size: medium
    depends_on: []
    description:
      "periodic-cron: resolve bob-cli-au by extending the reviewed tailnet root-cron
      installation to every daily, weekly, and monthly root entrypoint and verifying
      their complete executable chains."
  - id: backup-rotation
    title: Commit backup rotations when rsync reports only vanished files
    size: medium
    depends_on:
      - periodic-cron
    description:
      "backup-rotation: resolve bob-cli-1a in the root-owned backup.sh that every tier
      now sources, keeping genuine rsync failures fatal and proving rotations land
      again."
  - id: user-supervisor
    title: Restrict Bryan's user Supervisor bridge and the Multivisor web UI
    size: medium
    depends_on: []
    description:
      "user-supervisor: resolve bob-cli-av by moving the unauthenticated user
      multivisor-rpc bridge off the network and applying the chosen Multivisor web
      audience."
  - id: rfserver-port
    title: Decide and enforce rfserver's function-port boundary
    size: medium
    depends_on:
      - user-supervisor
    description:
      "rfserver-port: resolve bob-cli-aw by inventorying port 23401 clients, choosing
      the narrowest boundary that keeps them working, and verifying allowed and denied
      access."
  - id: ref-id-allocation
    title: Make new reference IDs follow stems and stay unique across ref/
    size: medium
    depends_on: []
    description:
      "ref-id-allocation: resolve bob-cli-8o so created references record their actual
      note stem and default collision walks cover every ref type directory."
proposed_by: bbugyi200.athena.bob-cli-an.land
proposed_with: claude/opus
parent_bead: bob-cli-an
decided_by: auto
create_time: 2026-10-23 12:41:08
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch2_athena_hardening_and_ref_ids.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch2_athena_hardening_and_ref_ids.md)
- **PARENT:**
  [202610/backlog_security_and_durability.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_security_and_durability.md)

# Backlog batch 2: Athena root, listener, and backup hardening plus reference ID allocation

## Scope and triage

The user requested a continuing backlog campaign of bounded epics. Batch 1
(`bob-cli-an`, plan `202610/backlog_security_and_durability.md`) closed `bob-cli-8u`,
`bob-cli-96`, `bob-cli-97`, `bob-cli-1g`, and `bob-cli-9o`. Its lander verified the live
host state and created four follow-ups from phase proposals and its own probes. It also
reopened `bob-cli-1a` with new evidence. This epic owns exactly these five task beads;
no phase may add a sixth:

| Rank | Task         | Reason for priority                                                                                                                                | Phase               |
| ---- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| 1    | `bob-cli-au` | Five daily, weekly, and monthly root cron entries still execute Bryan-writable symlinked code: a Bryan-to-root path with no sudo prompt.           | `periodic-cron`     |
| 2    | `bob-cli-av` | Bryan's user supervisord serves an unauthenticated control bridge on `0.0.0.0:8052`, reachable from apollo over the tailnet.                       | `user-supervisor`   |
| 3    | `bob-cli-aw` | rfserver's token-authed `/sendmail` port `23401` binds every interface and is reachable from the tailnet, possibly from the internet through DDNS. | `rfserver-port`     |
| 4    | `bob-cli-1a` | Backup rotations are discarded on rsync exit 24: hourly rotations land about every third run, and daily, weekly, and monthly rotations are stale.  | `backup-rotation`   |
| 5    | `bob-cli-8o` | `bob ref create` records ids that differ from note stems and walks collisions per type directory, leaving duplicate ids across `ref/`.             | `ref-id-allocation` |

Athena was compromised through a service account on 2026-10-17, so its remaining
privilege-boundary and network exposures come first. Backup recovery-point loss comes
next as content loss. Then comes the highest-ranked silent identity corruption from the
batch-1 candidate list. `bob-cli-8o` outranks `bob-cli-9a` on age and live-vault impact
(duplicate ids already needed manual repair). Related tasks remain independently owned:
`bob-cli-ax` (dead `clipster_server` scrape, small) and `bob-cli-8r` (detection
alerting). `bob-cli-9x` (vault-sync halt alerting), `bob-cli-9a` (relative `lib/` path
duplicates), and `bob-cli-1q` (PDF policy) also stay outside this batch. None is
silently absorbed here.

On 2026-10-23, after the batch-1 landing, a fresh enumeration found **155 unresolved
tasks: 151 ready and four snoozed** (`bob-cli-3o`, `bob-cli-6q`, `bob-cli-aa`,
`bob-cli-ah`), with none claimed or in progress. This is a snapshot, not a substitute
for the lander's fresh enumeration. Batch-1 spot checks found `bob-cli-v` (336 clippy
warnings remain) and `bob-cli-8n` (vault `AGENTS.md` still names Obsidian Sync) still
valid. No full obsolescence audit was rerun after batch 1's 2026-10-21 audit, so the
next lander owns that sweep.

Open epic plans at this snapshot were `bob-cli-28`, `bob-cli-3j`, `bob-cli-5w`,
`bob-cli-66`, `bob-cli-87`, `bob-cli-8b`, `bob-cli-9k`, `bob-cli-9k.8`, and
`bob-cli-at`. None owns this batch's tasks. Recheck ownership before implementation.

## Working rules and handoffs

Read the assigned task, this epic's finalized decisions, and phase dependencies through
`sase bead read`. Reproduce against the current tree and host before changing anything.
A task that has meanwhile been fixed can be closed with independently checked evidence
instead of reimplemented. Record baseline, chosen fix, validation, and remaining
constraints on the task and phase. Use `sase bead close` with `done`, `superseded`, or a
human-authorized `canceled` resolution; never hand-edit the store or runtime-owned
statuses. Phase workers leave task beads open for the lander unless their phase
description says otherwise; the lander closes them after rechecking evidence.

Use `/sase_repo` before accessing any other repository. Athena host source lives in the
`tailnet` project (`hosts/athena/supervisor/`, `hosts/athena/root-cron/`,
`hosts/athena/monitoring/`); Bryan's dotfiles are the linked `chezmoi` repo. Trace the
real source of any service, cron, or user Supervisor file before editing it. If it lives
in a repository that `sase repo open` cannot reach, record that constraint, and keep a
durable, reviewed copy plus install and rollback steps in tailnet. Batch 1's monitoring
phase lost its tailnet source to a workspace eviction until the lander recovered it from
a SASE rescue patch. **Every host phase must commit its tailnet source in the same turn
as the change, through the turn's final declaration.** Read sidecar evidence only
through `sase artifact read`.

The host phases run on Athena. First gather nonprivileged evidence, prepare exact files
and install/rollback argv, and validate what can be validated without root. Then use
`/sase_sudo` for the concrete privileged batch and its privileged verification. The
skill requires reviewed privilege; plan approval does not remove that requirement. Do
not disclose credentials or tokens in outputs; use suppressed-output mode for
secret-adjacent operations. Preserve SSH access and known working backups, monitoring,
mail, and service administration. A rejected gate is unfinished work, not a reason to
call the defect fixed. Use `/sase_monitor` for long commands or waits. Prove denial from
a real remote host (`ssh apollo`, per the tailnet memory note), not only from Athena's
own addresses: local connections to Athena's LAN or tailnet IP traverse loopback.

`periodic-cron` and `backup-rotation` are sequential because both edit the tailnet
root-cron installer and the root-owned `backup.sh` copy. After `periodic-cron`, every
backup tier sources that one copy, so the exit-24 fix lands once for all tiers.
`user-supervisor` and `rfserver-port` are sequential because rfserver is a program under
the same user supervisord and both may add firewall rules. The two host chains touch
disjoint files and can run in parallel. `ref-id-allocation` touches bob-cli only. Every
phase is bounded direct implementation work (`medium`); the listed outcomes and tests
are its implementation contract.

## Phase periodic-cron — bob-cli-au

Evidence (2026-10-23, as bryan): `/etc/cron.daily/daily_jobs`, `daily_backup`,
`/etc/cron.weekly/weekly_jobs`, `weekly_backup`, and `/etc/cron.monthly/monthly_backup`
are symlinks into `/home/bryan/db/bin/cron.jobs/cron.{daily,weekly,monthly}/`. The
hourly pair was already migrated by batch 1 into regular root-owned files that source
`/usr/local/lib/athena-cron/` (tailnet `hosts/athena/root-cron/`, commit `dbb7368`).

1. Inventory each entrypoint and its complete chain: every sourced helper, executable,
   interpreter, module, configuration able to select executable code, and PATH lookup.
   Record which steps truly need root, plus the exact outputs and schedules to preserve.
   Do not assume the jobs scripts match the hourly ones.
2. Keep unprivileged work unprivileged when coverage is unchanged. Install reviewed
   copies into the existing root-owned tree with pinned `PATH`, cleared
   `LD_PRELOAD`/`LD_LIBRARY_PATH`, and absolute helper paths. Extend
   `install_root_cron.py` and `verify_root_cron.py` rather than forking them. Keep their
   baseline guard, atomic install, manifest, rollback, and bryan/nobody denial probes,
   and cover every new entrypoint and parent directory.
3. Validate with scratch destinations or documented non-destructive modes; batch 1
   proved an accidental live run as bryan only failed safe by luck. Then install and
   verify through one reviewed `/sase_sudo` batch, including `run-parts --test` for each
   tier. Do not force destructive retention just to test scheduling.
4. Update the root-cron README for the full tier set and commit the tailnet source.

Acceptance: no root-executed daily, weekly, or monthly part is replaceable by `bryan`
through file, directory, PATH, symlink, sourced-helper, or module writes. Every tier is
still listed by its scheduler with the intended identity, with a tested rollback.

## Phase backup-rotation — bob-cli-1a

Evidence (2026-10-23 12:25 EDT): the newest `/mnt/hercules/backup/home/hourly` rotation
was 04:24, with `hourly-2..4` at 01:24, 23:24, and 20:24. The `TOO_SOON` guard is 30
minutes, so the gaps are discarded runs. The latest daily rotation was 2026-10-18,
weekly 2026-09-26, and monthly 2026-04-23. Batch 1's gated exercise saw rsync code 24 on
short-lived `~/.sase` scratch plus an rm staging race. `_backup()` deletes the staged
rotation on any nonzero rsync exit.

1. Reproduce from root mail or job output through a reviewed read, and confirm which
   tiers fail with exit 24 and which fail for other reasons. Stale daily, weekly, or
   monthly tiers with a different cause are evidence for the lander, not silent scope
   growth.
2. In the tailnet `lib/backup.sh`, commit the rotation when rsync's only failure is
   vanished source files (24), and record a warning in the job's error output. Keep
   every other nonzero code fatal. Decide exit 23 from the observed stderr and record
   why. Do not weaken the atomic `mv` chain, `--link-dest`, or `TOO_SOON` guard.
   Optionally exclude known-ephemeral agent scratch paths only when they are provably
   regenerable.
3. Test with an overridden scratch `BACKUP_DIR` and a fake rsync that returns 24 and 23
   (plus a real non-24 failure), then install through the reviewed installer and gate.
4. Verify that later scheduled runs advance the hourly rotation at least twice in a row
   and that a sample restore matches. Record tier freshness.

Acceptance: an rsync run that only lost vanished files commits its rotation; genuine
errors still discard and report; scheduled rotations advance on consecutive runs.

## Phase user-supervisor — bob-cli-av

Evidence (2026-10-23): Bryan's user supervisord runs
`[eventlistener:multivisor-rpc] command=multivisor-rpc --bind *:8052`, and `athena.conf`
runs `multivisor ... --bind *:8100`. A TCP connect from apollo to Athena's tailnet
address succeeds on both ports. The local `multivisor.conf` reaches the user bridge at
`localhost:8052` and the root bridge at `ipc:///tmp/supervisor-root-multivisor.sock`. It
also lists `hera` entries, but hera is absent from the current tailnet inventory.

1. Locate the authoritative source of `~/.config/supervisor/*` (chezmoi did not track it
   at batch 1's check). Inventory every consumer of 8052 and 8100, including any remote
   Multivisor instance and the `hera` entries.
2. Move the user bridge to an access-controlled Unix socket, as batch 1 did for root
   (`bryan:bryan 0700`), or to `127.0.0.1`. Update `multivisor.conf` without printing
   credentials. Remove dead `hera` entries only with evidence that the host is retired;
   otherwise record them.

> [!decision] multivisor_web_access = tailnet Accept 8100 from loopback and the tailnet
> interface only, dropping LAN and other interfaces for IPv4 and IPv6. Use a dedicated
> nftables table and unit in the style of `hosts/athena/monitoring` that does not depend
> on boot ordering of the tailnet address.

> [!decision] multivisor_web_access = loopback Bind 8100 to `127.0.0.1` and document the
> SSH-tunnel access path. Add a firewall rule only if the service cannot bind loopback.

3. Restart only the affected user Supervisor programs and keep the root administrative
   path intact. Verify authenticated Multivisor still controls both root and user
   Supervisors, and that remote probes from apollo are denied as decided. Commit the
   source and rollback steps to tailnet (and to the authoritative source repo when
   reachable).

Acceptance: no unauthenticated network path to Bryan's user Supervisor; the web UI is
reachable exactly by the decided audience; administration keeps working.

## Phase rfserver-port — bob-cli-aw

Evidence (2026-10-23): rfserver (`tools:rfserver` under Bryan's user supervisord, source
`~/db/bin/main/rfuncs/server.py`) listens on `0.0.0.0:23401`, which includes a
token-authenticated `/sendmail`. `RFSERVER_HOSTNAME=bbugyi.ddns.net` suggests clients
may use a public DDNS name. Its metrics port 9101 was already rebound to loopback in
batch 1.

1. Inventory 23401's clients from configs, process connections, and logs: local pymail,
   chezmoi-managed callers, other hosts, and anything resolving `bbugyi.ddns.net`. Check
   router forwarding and public reachability with nonintrusive probes; never print the
   token.
2. Choose the narrowest boundary that keeps the inventoried clients working: loopback,
   tailnet-only plus firewall, or an authenticated restricted proxy. If the inventory
   shows a real user trade-off (for example, an external client that must keep reaching
   it), prepare concrete alternatives and use `/sase_questions` rather than guessing.
3. Apply through the source repo (opened via `/sase_repo`) and a reviewed sudo gate for
   any firewall change, and update clients that need a new endpoint.
4. Verify that an authorized `/sendmail` call path still works without sending
   unsolicited mail, by inspecting existing delivery evidence or getting explicit
   authorization for one test send. Prove remote denial from apollo.

Acceptance: a documented 23401 client matrix and a verified boundary that keeps the
listed clients working and denies the rest.

## Phase ref-id-allocation — bob-cli-8o

Evidence (2026-10-23): `src/native/highlights_ref/stamp.rs::plan_default_target` and
`default_stem_occupants` probe only `xlib/<type>/`, `lib/<type>/`, and `ref/<type>/` for
the candidate stem. In the live vault, `harness_engineering` existed in both `ref/blogs`
and `ref/papers` until a manual repair. The `--output` path in `create.rs` keeps the
unsuffixed base stem as the id, so `BryanBugyi_CV_extended_v2..v5` all recorded
`id: BryanBugyi_CV_extended`. Governing decision:
`decisions:reference-ids-follow-file-stems` (an id equals its note stem; a `_N` suffix
only to avoid a duplicate across all of `ref/`).

1. Add failing regressions first: `--output` with a basename different from the
   generated stem, and same-stem creation across two type directories. Cover `clip.rs`,
   `collection.rs`, and `ingest.rs` callers of `plan_default_target`, not only
   `create.rs`.
2. Make the default collision walk treat an existing reference id anywhere under `ref/`
   (and its intake and library counterparts) as occupied. Record the id as the final
   note stem for every output path, including explicit `--output`. When an explicit
   `--output` stem collides with another type's id, follow the decision's suffix rule or
   refuse clearly; document whichever behavior the decision implies.
3. Do not re-key existing references; `bob ref rename` owns repairs. Keep `source_url`
   deduplication and the reading task's `^ref-…` block ID unchanged.
4. Run the focused highlights/ref tests, then `just check` once on the integrated
   bob-cli changes (use `/sase_monitor` if it runs long). Update `docs/ref.md` for the
   allocation contract and run `just docs-check`. Follow the daily release train: do not
   edit package versions or CHANGELOG, publish a manual release, or mutate the live
   library during validation.

Acceptance: created references always record their note stem as id, and no default
creation produces an id already used anywhere under `ref/`.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a sixth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and host state, including whether
another epic changed a shared service, cron, backup, or reference path. Confirm each
host phase's tailnet source is committed and byte-identical to the installed files.
Prove listener denials from apollo. Run appropriate integrated checks once; document
genuine preexisting failures with their existing task owners. Require actual
privilege-gate results, rotation-advance evidence, and ID regression results before
closing those tasks. Only the lander closes this epic after all its children are closed
and its normal landing obligations are met.

Because `sase plan propose` terminates the runner without running the commit finalizer,
the lander must commit every repository it changed (bob-cli, tailnet, the plans sidecar
status edit, and any other opened repo) with `/sase_git_commit`, which this approved
plan explicitly authorizes for that purpose, before proposing the successor.

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
   candidates include `bob-cli-8o`, `bob-cli-9a`, `bob-cli-8a`, `bob-cli-79`, and
   `bob-cli-al` for identity/write defects; heavily corroborated `bob-cli-6m`,
   `bob-cli-6v`, `bob-cli-6w`, and `bob-cli-7i` follow closely. This is guidance, not a
   preassigned batch, and concurrent work may obsolete it.
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
5. Preserve future prerequisites. The snoozed `bob-cli-3o`, `bob-cli-6q`, `bob-cli-aa`,
   and `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate
   has not arrived. Verify real prerequisites, not just wake timestamps. Work other
   actionable tasks first. If only gated tasks remain, the next bounded epic must own
   the readiness check and supported `/sase_monitor` or `/sase_questions` handoff needed
   to resume; do not abandon the chain, spin on fresh identical epics, or close a valid
   task merely to reach zero. Do not interfere with another worker's claim.
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
   truthfully; completing this first batch alone does not complete the backlog campaign.
