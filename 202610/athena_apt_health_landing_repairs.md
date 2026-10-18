---
tier: tale
size: medium
title: Repair Athena apt-health failure handling and finish bob-cli-8v landing
goal:
  Athena never reports an unverified apt refresh as source success, removed Plex
  packages are skipped, the GitHub CLI key is reproducible, and epic bob-cli-8v is
  closed with its approved verification and follow-up outcomes recorded.
proposed_by: bbugyi200.athena.bob-cli-8v.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-8v
create_time: 2026-10-18 10:29:00
status: wip
---

- **PARENT:**
  [202610/athena_plex_autoupdate_and_sandbox.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/athena_plex_autoupdate_and_sandbox.md)
- **BEAD:**
  [bob-cli-8v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8v/README.md)

# Remaining work only

Finish the landing of **bob-cli-8v**. This is one bounded coding task in the private
`tailnet` repository, followed by reviewed installation on Athena and the epic's
closeout. This tale has no separate land agent: **the coder must complete the closeout
in the same workflow**. Do not wait for this tale's own commit SHA, push, or CI result;
SASE commits after the final turn ends.

Read `sase bead read bob-cli-8v -r "Need the landing audit and triage outcomes"` and the
original plan with an audited read:
`sase artifact read plan:202610/athena_plex_autoupdate_and_sandbox.md "Need the approved acceptance and decisions"`.
The original plan file to mark done at the end is
**`202610/athena_plex_autoupdate_and_sandbox.md` in the plans repository** (resolve the
exact PLAN path from the bead; do not assume a host workspace location).

Use `/sase_repo` to open `tailnet` and read its `AGENTS.md`. Before editing, fetch and
safely integrate its current base branch; preserve other agents' changes. Also open
`plans` through `/sase_repo` before its final status edit. Read sidecar artifacts
through `sase artifact read`, not direct file reads.

## What the land audit already verified

Both epic phases, **bob-cli-8v.1** and **bob-cli-8v.2**, are closed. The original apt
task **bob-cli-8s** was closed by phase 1. The lander independently verified the sandbox
acceptance and closed **bob-cli-8t** on 2026-10-18. Recheck these states at closeout,
but do not repeat their entire implementation or restart Plex.

At `bob-cli` **b2a73b6** and `tailnet` **d95f153**:

- Read every child note and the original plan. Epic configuration commits are `1657987`
  (apt-health) and `d95f153` (sandbox and recovered probes).
- Reviewed the later, non-epic tailnet documentation commits `60d34ae` and `95c8a50`.
  Their prerequisites and restore ordering are valuable; their notes exposed the two
  collector defects below. Keep those documentation improvements.
- Later bob-cli commits cover capture `open_notes`, reference/note snapshots, Mermaid,
  keychain diagnostics, and documentation. None consumes or conflicts with the Athena
  configuration. Both local heads matched their fetched base.
- All ten tracked files under `athena/etc` and `athena/usr` matched the live installed
  bytes. All three recovered probe hashes matched their validation report, and their
  Python sources compile.
- Live Athena: nine apt sources up, no orphan list files, all seven `athena.apt` rules
  healthy, collector `Result=success`/`ExecMainStatus=0`, all three timers scheduled,
  and email notifications total 5 with every email failure counter 0.
  `promtool check rules`, `promtool test rules`, and `amtool check-config` pass.
- `/usr/bin/python3 -I` loading `/usr/bin/unattended-upgrade` confirms precisely the
  Plex origin plus the two Debian-Security patterns; `is_allowed_origin` accepts the
  Plex candidate. Use system Python for `python-apt`, not a user venv.
- The sandbox is installed with all approved extras, including explicit noexec on the
  `/mnt/hercules/plex` writable bind; the score is **3.3**, versus **9.2** before.
  `DevicePolicy=auto`, `DeviceAllow` empty. Plex is **disabled/inactive**, `MainPID=0`.
  Do not start, enable, or change its preferences for this tale.
- Completed gate transcripts independently confirm namespace checks passed, library
  counts stayed **7275/835**, no unexpected codec libraries or UPnP 32400 mapping, and
  zero sandbox-related log matches. Bryan answered **Played successfully** to forced
  lower-quality tailnet playback; restoration passed. Authenticated scan, server-side
  NVENC attribution, codec download and updater checks remain accurately documented N/A
  because the local token was absent; DLNA was disabled. The approved plan explicitly
  accepts N/A with reasons. These are not evidence that the unexercised features passed.

The approved decisions remain **plex_end_state=as_found**, **alert_delivery=email**,
**upgrade_scope=plex_and_security**, and **extra_hardening=extras**. Do not change them
or edit memory.

## 1. Repair collector health semantics and add focused regressions

Edit `tailnet:athena/usr/local/sbin/athena-apt-source-health` and add a small,
standard-library test suite in that repository. Import the script without running
`__main__`, mock subprocess/network/source discovery, and use temporary state/textfile
paths. Never poison the production state or metrics to test errors.

### Unattributed apt failures

Current `main()` considers a source healthy whenever no `Err:`/`W:`/`E:` line names it,
even if apt failed before checking sources. Reproduction on the unchanged source:

- Seed one discovered source with `last_success=100`, `consecutive_failures=3`.
- Mock `run_apt_update()` as either
  `("E: Problem executing scripts APT::Update::Pre-Invoke", 100)` or `("", 100)`.
- Run with `--now 1000000`, intercepting state/metric writes.
- It incorrectly produces `last_success=1000000`, `consecutive_failures=0`, and
  `source_up=1`, while `athena_apt_update_failed=1`.
- None of the existing alert rules consumes `athena_apt_update_failed`, so daily repeats
  can keep every source artificially fresh without an update-failure alert.

Compute source-attributed failures before mutating source history. Treat an unsuccessful
run (`apt_code != 0` or an `E:` line) with **no attributable source failure** as an
unverified refresh:

- Do not advance any source's last-success time or erase its failure history.
- Emit `source_up=0` for these unverified sources and preserve their existing
  consecutive-failure count; do not claim a source-specific failure was observed.
- For a newly observed source on this path, seed last success from the existing
  release-file mtime (or 0), with zero attributable failures.
- Continue to set the global failure metric. Successful recovery advances last success
  and clears failures normally.
- Keep the existing attributed-failure behavior: the real GitHub signature-failure
  fixture must still mark only that source down, increment its count, and preserve the
  success of other sources. Error attribution takes precedence over a `Hit:` or `Get:`
  line; those lines alone must not override a later verification error.
- Keep production apt-lock exhaustion as exit 75 without updating state/metrics or
  counting a source failure. Do not introduce extra apt calls or CLI flags.

Add **`AthenaAptUpdateFailing`**, warning/service `apt`, to
`athena/etc/prometheus/rules/athena-apt.rules.yml`, watching
`athena_apt_update_failed == 1` with `for: 1h`. It uses the existing email route. This
catches global failures even when source discovery produces no source series, and alerts
well within days. Do not alter the other seven thresholds.

Extend `athena/etc/prometheus/tests/athena-apt.rules.test.yml` with global failure
pending/firing/recovery tests. Unit regressions must cover both reproductions,
fresh/missing history, a successful recovery, a source-specific signature failure with
another healthy source, an empty discovered-source set with global failure, and
exhausted lock retries without sleeping. Check emitted metrics with promtool.

### Actual Plex installation status

`plex_installed_version()` currently queries only `${Version}`. dpkg retains a version
for a package in `config-files` state, so exit 0 and nonempty output do not establish
installation. This violates the original plan's skip-uninstalled contract.

Query status and version together (for example `${db:Status-Status}` and `${Version}`
with an unambiguous delimiter). Return a version only for actual `installed` state and a
nonempty version. Handle missing/malformed output and nonzero exits as absent. Preserve
existing `dpkg --compare-versions` behavior. Test installed, removed/config-files,
another noninstalled state, missing package, and malformed output. Verify
`collect_plex()` emits no Plex metrics and performs no upstream fetch when Plex is
absent. Do not install/remove a real package as a test.

## 2. Finish rebuild copies and integrate the updated documentation

The public keyring rotated by phase 1 is absent from tailnet's copies. Read the
non-secret live `/etc/apt/keyrings/githubcli-archive-keyring.gpg` on Athena, verify it
contains the unexpired signing fingerprint
**`7F38BBB59D064DBCB3D84D725612B36462313325`**, and copy it byte-for-byte to
`athena/etc/apt/keyrings/githubcli-archive-keyring.gpg` in tailnet. It currently also
contains the expired older public key; do not mistake that for a missing valid
replacement key. Never copy private keys or credentials.

Extend `athena/README.md` and `athena/apt-health-reference.md` to document the public
key, its provenance/fingerprint, and root:root 0644 installation under a root-owned 0755
`/etc/apt/keyrings`. Install it before the first restore-time apt refresh. Preserve the
requirement that the host's GitHub CLI apt source refers to that path; do not change
other sources or ship the expired backup as active. Binary keyrings cannot carry a
header comment; put the bead/purpose beside their runbook entry.

Update collector semantics, metric help text, the global-alert row and rule count, and
Plex installed-state documentation to match step 1. Preserve the post-epic
documentation's correct prerequisites, explicit first collection, Alertmanager restart
on config restore, and accurate limits. The Plex sandbox section, probe copies, and
runtime acceptance record stay intact.

## 3. Validate and install the bounded repair

Run the new offline collector tests, Python compilation, metric validation,
`promtool check rules`, `promtool test rules`, `amtool check-config`, and
`git diff --check`. Use the repository's verification recipe if one exists by then. Run
**`just check` in bob-cli** for the final file-change gate. Do not run
`just check-full`. If a command is long, use `/sase_monitor` with a continuation that
explicitly resumes this tale's installation/closeout; wait for the monitor handoff
command itself to exit. The tale remains responsible for closeout.

Use `/sase_sudo` for the privileged installation. Complete non-root editing, tests, and
review first. Stage non-secret files, then use explicit `install -o root -g root -m ...`
commands to place the collector (0755), rule file (0644), and rule tests (0644) in their
established paths. Execute only the installed, root-owned collector, never a
user-writable source path. The keyring is already correct on the live host; compare
hashes and do not rotate or rewrite it just to validate the repository copy.

Validate installed rules and Prometheus config, reload Prometheus, and run
`systemctl start athena-apt-source-health.service`. Check the oneshot using `Result` and
`ExecMainStatus`; `systemctl status` returning 3 for an inactive, successful oneshot is
not a failure. Keep the current timer and email routing.

After the gate, confirm:

- Installed/source hashes agree for the changed files and copied public key.
- A healthy collection has global failure 0, all configured sources up, and no orphan
  lists; Plex remains installed and its upstream check works.
- Prometheus loads all eight rules, including the new rule, with healthy evaluation.
- All three timers remain scheduled, email failure counters remain zero, and the
  existing Plex/security upgrade-origin selection is unchanged.
- Plex is still disabled/inactive, with the approved extras and score 3.3.

Record actual results, including any environmental limitation. Do not send a synthetic
production alert or alter credentials to re-prove email; historical delivered messages,
live notification metrics, and offline rule tests already establish that route. If
installation or required checks fail, fix them before closing. A new epic-caused defect
remains this tale's work.

## 4. Finish bob-cli-8v closeout (mandatory final step)

No further unrelated-task triage is pending. Before this tale was proposed the lander
used `/sase_new_task`, checked all statuses and the last week's tasks, reviewed all
active epic scopes, and recorded every outcome on **bob-cli-8v**:

| Proposing note                                                 | Outcome to include in the close note                                                                                                                                                                                                                              |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob-cli-8v.1 #3: root hourly cron scripts                      | **bob-cli-96**, bug/medium/ready. Independently confirmed writable targets; distinct from supervisor/venv bob-cli-8u and rsync handling bob-cli-1a; related links recorded.                                                                                       |
| bob-cli-8v.1 #4: wildcard monitoring listeners                 | **bob-cli-97**, bug/large/ready. Confirmed 9090/9091/9100 wildcard binds, not internet reachability; related to 8r/8u.                                                                                                                                            |
| bob-cli-8v.1 #5: notify 8r of email route                      | Completed integration note on **bob-cli-8r**. No new task because the delivery reuse information is already supplied. Canceled **bob-cli-18** also has resolution evidence; no reopen or +1.                                                                      |
| bob-cli-8v.2 #1 and #2: authenticated API/NVENC/codec coverage | Consolidated into **bob-cli-98**, feature/large/ready, related to 8t with validation artifact. Missing local credentials and disabled features predate this epic; accepted N/A coverage is not a reproduced sandbox regression. Declined a duplicate second task. |
| bob-cli-8v.2 #1: client playback pending                       | Superseded by the completed **Played successfully** answer. Client/resolution/hardware details remain unknown; no repeat of the accepted fallback is required.                                                                                                    |

Re-read the epic and both phase notes for any intervening discoveries; confirm
**bob-cli-8s**, **bob-cli-8t**, and both phases remain closed. Check post-audit drift on
bob-cli and tailnet again, applying any relevant integration.

1. Run `sase bead epic-symbols bob-cli-8v`. It had no entries during the audit. Resolve
   any new entry by wiring, privatizing, adding an appropriate non-test pragma, or
   deleting it. Re-key only when a **still-open later bead** really needs the exemption.
   Do not leave stale entries on this epic or its phases.
2. Close normally with `sase bead close bob-cli-8v --note "<verification>"`. Write a
   concrete note covering the original phase/source/commit audit, honored decisions,
   later-docs integration, the two collector repairs and key copy, actual installed/test
   results, accepted sandbox limitations, and **every triage outcome above**. Both
   original tasks are closed; do not wait for this tale's own commit.
3. Run `just symvision` if available to confirm a clean whitelist. Neither bob-cli nor
   tailnet had that recipe at the audit; if still unavailable, record that explicitly.
   If close rejects leftover symbols, resolve them and retry. If it reports unfinished
   phases, finish/reopen the named work. Never use `--force` merely to make a successful
   landing advance. A deliberate canceled or superseded outcome requires its actual
   reason and is not the expected path.
4. Set **`status: done`** in the frontmatter of the original epic plan
   **`202610/athena_plex_autoupdate_and_sandbox.md`** in the opened plans repo, using
   the current PLAN path returned by the bead.
5. Run `sase bead read bob-cli-8v -r "Need the parent link"`. The audited epic had **no
   parent_bead**; when unchanged, finish normally. If a parent link was added, do not
   guess: a phase parent closes only after its required work is verified; a plan parent
   requires its prior landing note, every descendant and linked plan, and post-child
   drift rechecked before a normal close, symbol cleanup, symvision, and its plan status
   update. Stop at the first incomplete or ambiguous parent, note the concrete blocker
   there, and report it. Never force a successful nested landing.
6. Submit `/sase_final` for every changed repository, including tailnet and the plans
   repository. Let the host commit the completed work after the turn ends; do not
   manually commit or defer this turn's own files.
