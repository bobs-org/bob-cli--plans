---
tier: epic
title: 'Backlog batch 1: Athena security and vault durability'
goal: Close five prioritized task beads by removing Athena privilege-boundary exposures,
  restricting monitoring access, preserving ignored vault content, and stabilizing
  annotation identities. The lander continues the backlog through further epics of
  at most five phases and five task beads until every project task bead is closed.
decisions:
  legacy_migration_backup:
    ask: Preserve legacy migration reports and helper sources in the vault Git backup?
    choices:
      preserve: Back up migration reports and helper sources along with other authored
        vault content
      exclude: Leave migration material excluded; back up the remaining authored vault
        content
    default: preserve
    why: These recovery materials currently fall outside Git backup; caches remain
      excluded.
    answer: preserve
phases:
- id: supervisor
  title: Secure Athena supervisor control and its executable chain
  size: medium
  depends_on: []
  description: 'supervisor: resolve bob-cli-8u with restricted authenticated control,
    protected credentials, root-owned dependencies, and live verification.'
- id: root-cron
  title: Remove user-writable code from root hourly jobs
  size: medium
  depends_on:
  - supervisor
  description: 'root-cron: resolve bob-cli-96 while preserving hourly jobs and backups
    and verifying their complete executable chains.'
- id: monitoring
  title: Restrict monitoring endpoints to verified clients
  size: medium
  depends_on:
  - root-cron
  description: 'monitoring: resolve bob-cli-97 after inventorying scrapes, pushers,
    dashboards, and firewall policy; retain apt alerts and email delivery.'
- id: vault-content
  title: Preserve ignored vault content off host
  size: medium
  depends_on:
  - root-cron
  description: 'vault-content: resolve bob-cli-1g with an inventory, narrow ignore
    exceptions, durable copies, and a restore check, honoring the migration-material
    decision.'
- id: highlight-identity
  title: Preserve highlight IDs across annotation insertion
  size: medium
  depends_on: []
  description: 'highlight-identity: resolve bob-cli-9o by preserving existing annotation
    identities before allocating new IDs, including folded annotations and duplicate-text
    cases.'
proposed_by: bbugyi200.athena.12v
proposed_with: codex/gpt-6-astra
decided_by: reviewer
decided_via: tui
create_time: 2026-10-21 12:58:22
status: done
bead_id: bob-cli-an
---

- **PROMPT:** [prompts/202610/backlog_security_and_durability.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_security_and_durability.md)
- **BEAD:** [bob-cli-an](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/README.md)

# Backlog batch 1: Athena security and vault durability

## Scope and triage

The user requests a continuing backlog campaign, not one unbounded implementation. This
epic owns exactly these five task beads; no phase may add a sixth:

| Rank | Task         | Reason for priority                                                                          | Phase                |
| ---- | ------------ | -------------------------------------------------------------------------------------------- | -------------------- |
| 1    | `bob-cli-8u` | Root Supervisor control is exposed through an unauthenticated bridge and user-writable code. | `supervisor`         |
| 2    | `bob-cli-96` | Hourly root cron executes two user-writable entrypoint chains.                               | `root-cron`          |
| 3    | `bob-cli-97` | Monitoring listeners expose data and ingestion beyond a verified client boundary.            | `monitoring`         |
| 4    | `bob-cli-1g` | Authored vault content falls outside the only active cross-host sync/backup channel.         | `vault-content`      |
| 5    | `bob-cli-9o` | Inserting annotations changes unchanged highlights' addresses and breaks task backlinks.     | `highlight-identity` |

The security items take precedence because Athena recently suffered a service-account
compromise and these remaining boundaries still fail read-only checks. Vault durability
and reference identity come next because they can lose content or corrupt durable links.
Test failures, performance improvements, and optional product additions follow in later
batches. Related tasks remain independently owned: `bob-cli-8r` is detection/alerting,
`bob-cli-1q` is general PDF policy, `bob-cli-8o` is reference-note ID allocation, and
`bob-cli-9x` is cross-host sync-halt alerting. None is silently absorbed here.

On 2026-10-21 the audited enumeration found **165 unresolved tasks**, including four
snoozed tasks and no claimed/in-progress task workers. Twelve were closed with evidence
notes, leaving **153 unresolved tasks: 149 ready and four snoozed**. This is a snapshot,
not a substitute for the lander's fresh enumeration.

| Closed tasks                             | Resolution and evidence                                                                                                                                                                                                                                                                                                                |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob-cli-81`, `bob-cli-9w`, `bob-cli-49` | Done: `67a26ea80a759b293188f17504a064385ee850c3` repairs ignored quarantine paths and lock-interrupted fast-forwards, adds a repeated-merge halt, and removes broken notification calls. Current source and regression tests were reviewed; no new Rust suite run is claimed. Historical quarantine deletion remains separately gated. |
| `bob-cli-7f`                             | Done: Mac commit `4da688e` repairs the three invalid multiline fixtures; current `swiftc -frontend -parse` succeeds.                                                                                                                                                                                                                   |
| `bob-cli-7o`                             | Done: Mac commit `c2f9999` supplies the SDK to all direct typechecker paths; shell syntax check succeeds. No new live macOS run is claimed.                                                                                                                                                                                            |
| `bob-cli-43`                             | Done: all current CaptureCore sources typecheck on Athena's Swift 6.0.3; `hintTokens` now uses incremental typed appends. Unrelated warnings remain.                                                                                                                                                                                   |
| `bob-cli-6e`                             | Done: Athena SSH config is no longer a dangling symlink, aliases resolve, and live vault-sync reports success at `2026-10-21T12:42:56-04:00` with no error or halt.                                                                                                                                                                    |
| `bob-cli-57`                             | Done: audited glossary already describes watcher refresh, 60-second safety polling, and 15-second fallback.                                                                                                                                                                                                                            |
| `bob-cli-63`, `bob-cli-64`, `bob-cli-75` | Superseded: accepted `decisions:unblocked-tasks-are-offered` and `glossary:successor-offer` cover the current behavior; automatic successor linking is retired.                                                                                                                                                                        |
| `bob-cli-4n`                             | Superseded by `bob-cli-6c`: its reference-note containment correction is part of the broader parent-residence correction. The narrower concern was transferred to the surviving bead.                                                                                                                                                  |

The first plan's inspection used bob-cli `2f0751d4`, bob-mac-capture `d73d943`,
bob-plugins `9202d3e`, and vault `cbaf0734`. Open plan beads at audit time were
`bob-cli-28`, `bob-cli-3j`, `bob-cli-5w`, `bob-cli-66`, `bob-cli-87`, `bob-cli-8b`,
`bob-cli-9k`, `bob-cli-9k.8`, and `bob-cli-ab`. They are not duplicate tasks and are not
closed or taken over by this campaign. Recheck ownership before implementation.

## Working rules and handoffs

Read the assigned task, this epic's finalized decisions, and phase dependencies through
`sase bead read`. Reproduce against the current tree before changing anything. A task
that has meanwhile been fixed can be closed with independently checked evidence instead
of reimplemented. Record baseline, chosen fix, validation, and remaining constraints on
the task and phase. Use `sase bead close` with `done`, `superseded`, or a
human-authorized `canceled` resolution; never hand-edit the store or runtime-owned
statuses.

Use `/sase_repo` before accessing any other repository. Open the configured `chezmoi`
repo, `tailnet` project, and `gh:bobs-org/bob` as needed, and use only returned paths
for repository reads/writes. The opened tailnet checkout currently has no deployment
code; do not assume scripts already exist there. Trace actual service/cron source
ownership and open that source through the same skill before inspecting it. Keep source
and repeatable installation/rollback instructions in the appropriate opened repository.
Existing source in a home directory is not authorization to bypass repository access.
Read sidecar evidence only through `sase artifact read`.

The host phases run on Athena. First gather nonprivileged evidence, prepare exact files
and install/rollback argv, and validate what can be validated without root. Then use
`/sase_sudo` for the concrete privileged batch and its privileged verification. The
skill requires reviewed privilege; plan approval does not remove that requirement. Do
not disclose configuration credentials in outputs. Secret-adjacent operations must use
the skill's suppressed-output mode. Preserve SSH access and known working backups,
monitoring, and service administration. A rejected gate is unfinished work, not a reason
to call the defect fixed. Use SASE handoffs/monitoring for long commands or waits.

The host phases are sequential because they may share service managers, root-owned
install locations, and backup infrastructure. `vault-content` follows `root-cron` so its
restore proof can trust the repaired backup chain. `highlight-identity` touches bob-cli
only and can run independently. Every phase is bounded direct implementation work
(`medium`); the listed outcomes and tests are its implementation contract.

## Phase supervisor — bob-cli-8u

Audit evidence: Athena still listens on `0.0.0.0:8051`; the Supervisor configuration is
root-owned mode `0644`. The task identifies a root eventlistener launched from Bryan's
pipx environment, an unauthenticated zerorpc interface, and readable XML-RPC
credentials.

1. Inventory Supervisor programs, users, include files, executable/interpreter/import
   paths, and actual Multivisor consumers. Record the control operations clients need.
   Establish a known-working protected administrative path before restarting anything.
2. Remove the unauthenticated root-control transport. Prefer the existing authenticated
   Supervisor interface or an access-controlled Unix socket for retained administration;
   retire the bridge if it has no legitimate consumer. If a consumer relies on it,
   migrate that consumer to an authenticated, restricted supported control path before
   removal. Binding unauthenticated root RPC only to loopback is insufficient because
   compromised local service accounts could still call it.
3. Install retained privileged code and its complete interpreter/import chain in a
   root-owned location whose ancestors and dependencies are not user-writable. Remove
   unnecessary root execution where the actual privilege requirements permit it. Protect
   the configuration and included secret files; rotate the exposed credentials and
   update legitimate clients without printing either credential.
4. Prepare a minimal reviewed installation with service-specific validation and a
   rollback that preserves administrative access. Avoid broad Supervisor restarts when a
   targeted reload suffices. Capture redacted before/after evidence.

Acceptance: no unauthenticated path to the root Supervisor API from ordinary local
accounts, LAN, or tailnet clients; no root-executed user-writable dependency; secret
files cannot be read by an unprivileged service account; legitimate administration and
previously running programs remain healthy. Test denial as well as successful
administration. If compatibility genuinely requires a new user choice, prepare the
concrete alternatives and use `/sase_questions`; do not quietly disable a used workflow.

## Phase root-cron — bob-cli-96

Audit evidence: resolving `/etc/cron.hourly/hourly_jobs` and `hourly_backup` still leads
to `bryan:bryan 0755` files in the legacy `db/bin/cron.jobs/cron.hourly/` tree. The
defect is the whole executable chain, not just symlink ownership.

1. Inventory both entrypoints and every sourced helper, executable, interpreter, module,
   configuration capable of selecting executable code, and environment/PATH lookup.
   Identify which steps truly require root and the exact outputs/schedules to preserve.
2. Keep unprivileged work unprivileged. For required privileged jobs, install reviewed
   code and dependencies into a root-owned, non-user-writable tree with explicit safe
   executable paths and a controlled environment. Repository source may remain
   developer-owned; root executes only the reviewed installation, never that source.
3. Add a repeatable deployment check that follows symlinks and verifies owner/mode of
   every executable dependency and parent directory. Prepare the sudo batch only after
   source review and a diff of the proposed install are complete.
4. Validate job behavior with a safe scratch destination or documented non-destructive
   mode, then verify the installed scheduler path and backup success/restore evidence.
   Do not run destructive retention/pruning merely to test scheduling. Preserve the
   canceled `bob-cli-1a` scope decision; rsync exit-24 policy is not reopened here.

Acceptance: the `bryan` account cannot replace any root-executed part through file,
directory, PATH, symlink, sourced-helper, or module writes. Both hourly workflows still
run on schedule with the intended identity and usable output. Record a tested rollback.

## Phase monitoring — bob-cli-97

Audit evidence: Athena still has wildcard listeners on ports 9090, 9091, and 9100. This
establishes binding, not public-internet reachability. The three distribution systemd
units are active. A redacted configuration read shows local static scrapes of Prometheus
(9090), pushgateway (9091), node exporter (9100), rfserver (9101), and clipster_server
(9102), with Alertmanager at localhost:9093 and no dynamic discovery in those jobs.
Remote pushers/dashboard consumers and firewall rules still need verification; local
scrape targets alone do not prove local-only use.

1. Inventory actual Prometheus scrape jobs, push clients, dashboard consumers, listener
   service definitions, tailnet addresses, IPv4/IPv6 firewall rules, and the existing
   apt-health/Alertmanager route. Produce an endpoint-to-client matrix backed by configs
   and read-only probes; do not infer that every client is local.
2. Apply the narrowest supported boundary preserving those clients: loopback for local
   consumers, specific tailnet binding plus ACL/firewall restrictions for remote ones,
   or an authenticated restricted proxy where the service requires it. Restrict both IP
   families and direct backend access; a public-facing proxy must not leave a bypassable
   wildcard backend. Keep any unauthenticated ingestion endpoint inaccessible to
   unintended clients.
3. Validate configurations, then install the staged change through `/sase_sudo` with
   rollback and a preserved SSH path. Update actual client endpoints where necessary.
4. Verify legitimate scrape targets remain up, an authorized pusher still works, and
   dashboard reads still work. Probe denied access from a host/interface outside the
   intended client set. Confirm apt-health metrics and the existing Alertmanager/email
   route configuration remain intact. Do not send unsolicited test emails; inspect
   existing delivery evidence or separately obtain explicit authorization for a send.

Acceptance: a documented, verified access matrix replaces wildcard exposure, with
successful authorized probes and denied unauthorized probes. This phase does not add the
separate detection rules in `bob-cli-8r` or a second alert-delivery system.

## Phase vault-content — bob-cli-1g

Current vault policy still starts with `*`, allows a fixed set of extensions, and omits
the reported `.zot`, `.xmind`, `.tsv`, non-Obsidian `.json`, helper `.py`, `.puml`, and
textbundle `.markdown` classes. The opened current vault tracks none of the queried
`.zot`, `.xmind`, `.tsv`, or `.puml` paths. The old counts (96, later 86) are
historical; a fresh clone does not contain ignored local files and cannot prove today's
count.

1. Inventory actual candidate content on each available host through the supported
   repository/host workflow, keeping paths, sizes, hashes, and provenance in a private
   report. A cloned Git tree alone is insufficient. Preserve distinct host variants
   before reconciling them; do not replace one machine's ignored copy with another.
   Classify authored content, archival recovery material, regenerable outputs, local
   state, and secrets. Bound the inventory to this bead's reported extension gap.
2. Add narrow directory/extension allow rules for authored content such as mind maps,
   templates, diagrams, text files, and textbundle metadata/body. Preserve intentional
   exclusions for custom plugin builds, caches/bytecode, editor state, `lit_review/`,
   `xlib/`, and the over-limit PDFs. Do not solve this with a blanket unignore.

> [!decision] legacy_migration_backup = preserve Include the reported `_meta/migration/`
> reports, inventory, and helper sources after checking for secrets and regenerable
> junk. Preserve their paths and bytes.

> [!decision] legacy_migration_backup = exclude Keep `_meta/migration/` excluded and
> record that accepted exclusion in the task's disposition. Do not delete it or claim it
> has acquired an off-host backup. All other authored-content coverage remains required.

3. Exercise candidate rules against scratch filenames with `git check-ignore`, including
   negative cases for caches, plugin output, private local state, and oversized files.
   Stage only the reviewed policy and selected content in the skill-opened repository.
   If live ignored files must be brought into that checkout, use the approved repository
   workflow; never substitute direct reads of a forbidden sibling/live checkout. If that
   workflow cannot expose ignored source content, request the needed access through
   `/sase_questions` and preserve the task as unfinished; do not claim that an empty
   clone inventory proves the live content is safe.
4. Publish through the repository's authorized SASE commit/deployment flow and verify
   the resulting remote tree contains the intended bytes. Confirm synchronization to
   another available host or restore from the remote into isolated scratch and compare
   hashes. Update `docs/vault-git-sync.md` for the actual coverage, then run the docs
   gate.

Acceptance: every included candidate has a verified off-host copy and a successful
restore/hash check; every remaining exclusion has a concrete justification. A Gitignore
edit without the content upload is incomplete. Unreachable host-only variants remain
explicitly unresolved until checked. Do not broaden into general PDF storage policy.

## Phase highlight-identity — bob-cli-9o

The current `src/native/highlights_ref/sidecar_render.rs::annotation_block_id` hashes
`ordinal_on_page`. Both highlight insertion and the now-shipped underline/continuation
folding paths can change ordinals. The renderer calls this helper in normal rendering,
fold membership, absorbed IDs, and diagnostics; fixing one call site is insufficient.

1. Add regression fixtures reproducing the reported sequence: scan a page with three
   highlights, insert another above them, rescan, then insert an absorbed underline
   above its host and rescan. Existing addresses and annotation-task backlinks must
   continue to name live quotes with no false removed-highlight entries.
2. Plan IDs once per reference before rendering. Reuse parsed current managed blocks
   (`region.rs::parse_managed_region`, `RegionBlock`, and the note's managed region) to
   preserve existing IDs for provably unchanged annotations. Match by source, kind,
   page, and normalized semantic content rather than absolute annotation ordinal.
   Presentation-only emphasis/return-link decoration must not change that identity.
   Folded groups must resolve their head and absorbed members through the same plan.
3. Allocate new IDs from the stable content key, with a deterministic collision
   disambiguator only within identical-key groups. Reuse existing group members one to
   one before allocating extras; do not globally renumber duplicate text when unrelated
   annotations are inserted. Reserve existing live and removed IDs against accidental
   reuse, and never silently drop a distinct annotation on a hash collision.
4. Make adoption conservative: existing ordinal-derived IDs remain valid addresses; no
   vault-wide rewrite or new user-maintained identity field is needed. Preserve authored
   text and existing removed-highlight anchors. For a legacy/folded group whose identity
   cannot be safely established, return a write-free diagnostic for that note instead of
   guessing, changing backlinks, or tombstoning unchanged content. Do not use fuzzy text
   matching that could assign a quote another quote's address.
5. Thread the ID plan through rendering, fold absorption, continuation/underline
   diagnostics, and annotation-derived task links. Preserve image hashing and existing
   suppression semantics. Test duplicate text on one page, same text across pages and
   kinds, insertion/deletion/reordering, comment edits, real content removal, rescan
   idempotence, legacy adoption, folded groups, and ambiguous input without writes.
6. Run focused highlights/ref regression tests, then `just check` once on the integrated
   bob-cli changes; use `/sase_monitor` for a long run. Review baseline failures rather
   than weakening assertions. Update the identity behavior contract in the relevant
   highlights docs and run `just docs-check`. Follow the daily release train: do not
   edit package versions or CHANGELOG, publish a manual release, or silently migrate the
   live library during validation.

Acceptance: the regression sequences retain all preexisting live IDs and backlinks, new
annotations receive unique IDs, actual removals retain legitimate tombstones, and
repeating a scan is byte-stable. Fixture validation must include the current fold paths,
not only plain unmerged highlights.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a sixth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and host state, including whether
another epic changed a shared service or annotation path. Run appropriate integrated
checks once; document genuine preexisting failures with their existing task owners.
Require actual privilege-gate results, actual off-host content/restore evidence, and
identity regression results before closing those tasks. Only the lander closes this epic
after all its children are closed and its normal landing obligations are met.

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
