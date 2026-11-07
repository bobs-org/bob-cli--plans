---
tier: epic
title: Crontabs as code through chezmoi, with first-class agent access
goal: 'One committed cron: block in the chezmoi repo is the single source of truth
  for every machine''s user crontab; chezmoi update -a converges each machine''s own
  marked block without touching other lines, and agents can read and write any machine''s
  crontab from any machine through cronctl, a /crontab skill, and a global memory
  note.'
phases:
- id: render
  title: Cron data schema, partials, and per-machine rendering
  depends_on: []
  size: medium
  description: 'render: add the cron: schema, validating chezmoi partials, the per-machine
    managed.crontab and retire.crontab targets, and render tests on chezmoi v2.13.1
    and the latest release.'
- id: cronrun
  title: Machine-agnostic job runner
  depends_on: []
  size: small
  description: 'cronrun: add ~/bin/cronrun, which derives the job environment at run
    time, guards overlaps, logs, records status, mails only failures, and offers a
    side-effect-free --dry-run.'
- id: sync
  title: Local reconciler, hook, and offline cronctl commands
  depends_on:
  - render
  size: medium
  description: 'sync: create cronctl with list, render, and check plus the marked-block
    sync reconciler, and the run_onchange hook that runs it when this machine''s block
    changes.'
- id: remote
  title: Fleet commands for status, diff, apply, logs, run, and edit
  depends_on:
  - sync
  - cronrun
  size: medium
  description: 'remote: add SSH-streamed fleet commands so any machine can read live
    state and converge any machine to a pushed revision without a bootstrap step.'
- id: agent-docs
  title: Crontab skill, memory note, README, and alias cleanup
  depends_on:
  - remote
  size: small
  description: 'agent-docs: teach agents the read and write flow through a global
    /crontab skill and reference memory note, document the human flow, and retire
    legacy cron aliases.'
- id: migrate
  title: Move athena and Mac jobs into the data and cut over
  depends_on:
  - remote
  - cronrun
  - agent-docs
  size: medium
  description: 'migrate: move the live athena and Mac jobs into cron: data with an
    atomic cutover, update bob-cli docs, and converge the fleet through a post-commit
    apply.'
decisions:
  cutover:
    ask: 'Swap legacy crontab lines atomically via a temporary cron.retire: list that
      sync removes with the block?'
    choices:
      retire: Atomic, unattended cutover on athena and the Mac; legacy lines sit in
        the file until cleanup
      supervised: No retire feature; manual cutover on athena and later the Mac, with
        brief skipped or doubled runs
    default: retire
    why: Agents can't act after their own commit lands, and the Mac is often asleep
    answer: retire
  mac_schedule:
    ask: Should the Mac jobs adopt the staggered schedule and --retry-timeout 120
      that docs/vault-git-sync.md documents?
    choices:
      documented: Stagger at :00/:05/:10 with --retry-timeout 120 as documented; changes
        what runs on the Mac
      live: Keep today's three */15 lines with no retry timeout, and rewrite the docs
        to match
    default: documented
    why: The docs explain the staggering; the live table looks like accidental drift
    answer: documented
  log_dir:
    ask: Should cronrun write job logs to ~/.local/state/cron/<job>.log instead of
      the legacy /var/tmp paths?
    choices:
      state: One size-capped log per job under ~/.local/state/cron, read via cronctl
        logs; docs updated
      var_tmp: 'Keep each legacy /var/tmp filename through a new optional per-job
        log: field'
    default: state
    why: Job-keyed logs are what cronctl logs and status can find on every machine
    answer: state
  crontab_memory_note:
    ask: 'Create a global reference memory note crontab.md making cron: data the only
      source of truth for cron jobs?'
    memory:
    - crontab.md
    default: true
    requested: Make sure agents have first-class support (i.e. easy ability to read/write
      my crontab on any machine from any machine) for this new and improved way of
      managing my crontabs.
    answer: true
  tailnet_memory_link:
    ask: Add a one-line [[crontab.md]] cross-link to tailnet.md, since cronctl addresses
      machines by these aliases?
    memory:
    - tailnet.md
    default: true
    requested: Review the crontab_as_code_via_chezmoi.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file
    answer: true
proposed_by: bbugyi200.athena.research.5w.linker.w1
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-07 12:43:36
status: wip
bead_id: bob-cli-ib
---

- **PROMPT:** [prompts/202611/crontab_as_code.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/crontab_as_code.md)
- **BEAD:** [bob-cli-ib](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ib/README.md)

# Crontabs as code through chezmoi, with first-class agent access

## Goal

Replace per-machine `crontab -e` editing with one checked-in source of truth: a `cron:`
block in the chezmoi repo's `home/.chezmoidata.yaml`. On each machine,
`chezmoi update -a` converges that machine's user crontab. Agents get a documented tool
(`cronctl`), a global `/crontab` skill, and a global reference memory note, so they can
read and write any machine's crontab from any machine.

This plan adopts the recommended solution and every requirement adjustment (ADJ-1 to
ADJ-13) from the research report `crontab_as_code_via_chezmoi.md` in the research
sidecar. Workers may read it with
`sase artifact read file:explicit:445fd652fdec1c9399f479d0 "<why>"`. This plan is
self-contained, and where it differs from the report, this plan wins.

## Where the work happens

Almost all work happens in the **chezmoi** linked repo (`bbugyi200/dotfiles`, a public
repo). Every phase worker opens it first with `sase repo open chezmoi -r "<why>"`, reads
its `AGENTS.md`, and uses the printed path for all reads and writes. Paths below are
relative to that checkout's `home/` directory (the repo's `.chezmoiroot`) unless they
start with `tests/` or name the repo root. The `migrate` phase also edits bob-cli's own
`docs/`. Commit every changed repo through the `/sase_final` declaration.

## Facts verified while planning (2026-11-07)

- **athena** (Debian, `America/New_York`): chezmoi **v2.13.1** at
  `~/.nix-profile/bin/chezmoi`, the oldest in the fleet. Its live crontab holds the
  Debian boilerplate header, 5 env lines (`DISPLAY=:0`,
  `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus`, `HOME=/home/bryan`,
  `MAILTO=...`, `XDG_RUNTIME_DIR=/run/user/1000`), commented-out `tman` and `init-bg`
  jobs, `bob nightly` at `30 3 * * *`, `pass_git_sync` at `0 * * * *`, and an
  agent-installed watchdog line tagged `# sase-epic-watchdog:sase-1r2`. That watchdog
  line must survive untouched.
- **apollo** (Debian, `Etc/UTC`): no crontab. It has two chezmoi binaries:
  `~/bin/chezmoi` (v2.70.2) and `~/go/bin/chezmoi` (dev). Neither is on a
  non-interactive SSH `PATH`. Its source checkout is clean but behind `origin`.
- **mac** (SSH user `bbugyi`, chezmoi hostname `Kellys-MacBook-Pro`, macOS 26.5,
  `America/New_York`): chezmoi **v2.52.1** at `/opt/homebrew/bin/chezmoi`, also not on
  the non-interactive SSH `PATH`. It has no `flock` and bash 3.2 at `/bin/bash`. Its
  live crontab has three `*/15` jobs (`~/bin/maybe_bob_highlights_sync -w`,
  `~/.cargo/bin/bob projects sync`, and `~/.cargo/bin/bob task reconcile`), each
  appending to a `/var/tmp/*.log`. Its `bob task reconcile` accepts `--retry-timeout`.
  Under a cron-like `env -i`, `bash -c '. ~/.profile'` exits 0 and puts `bob`,
  `chezmoi`, and `maybe_bob_highlights_sync` on `PATH`. The Mac is often offline, so
  treat SSH to it as best-effort.
- `~/.profile` is **bash**, not POSIX sh, so jobs must run under bash.
- On chezmoi v2.13.1, these all work: `.chezmoidata.yaml` at the source root;
  `.chezmoitemplates` partials called as `template "x" (dict ...)`; `fail`, `hasKey`,
  `regexMatch`, `toJson`, `sha256sum`, `include`, and `output`; and
  `chezmoi --source <repo-root> execute-template`, which honours `.chezmoiroot`.
  **`includeTemplate` does not exist**, so a template can't capture a partial's output
  in a variable. Partials must render whole outputs.
- Sprig's `squote` doesn't escape embedded `'`. The renderer must quote commands itself.
- SASE commits push to `origin`. Agents can't run commands after their own commit lands,
  except through a staged `sase monitor start` that the host starts after finalization.
  Don't assume the chezmoi project's `commit_hooks.after` (`chezmoi update -a --force`)
  fires for commits made from another project's linked checkout.
- Repo conventions: bashunit tests in `tests/bash/*_test.sh` run via `just test-bash`.
  Template tests render with `chezmoi --config <empty> --source <repo-root>`; see
  `tests/bash/hammerspoon_restart_hook_test.sh`. CI (`.github/workflows/ci.yml`)
  installs the **latest** chezmoi on Ubuntu, while athena runs v2.13.1, so templates
  must work on both. Markdown goes through
  `prettier --prose-wrap=always --print-width=88`. Run `just lint` and `just test` in
  the chezmoi checkout before finishing. If a toolchain that `just test` needs is
  missing locally, run at least `just test-bash` and `just lint-md`, and say so; CI runs
  the rest.
- Until a machine applies the new source, its `~/bin/cronctl` is missing or stale. Run
  the checkout's own script (`<checkout>/home/bin/executable_cronctl`) during
  implementation and in post-commit monitors.

## Shared contract

Every phase builds against this contract. Don't change a name or format here without
updating every phase that uses it.

### Files (chezmoi repo, under `home/`)

| Path                                                      | Deployed to                      | Owner phase                     |
| --------------------------------------------------------- | -------------------------------- | ------------------------------- |
| `.chezmoidata.yaml` (`cron:` key)                         | template data                    | render (schema), migrate (jobs) |
| `.chezmoitemplates/cron-*` partials                       | —                                | render                          |
| `dot_config/cron/managed.crontab.tmpl`                    | `~/.config/cron/managed.crontab` | render                          |
| `dot_config/cron/retire.crontab.tmpl`                     | `~/.config/cron/retire.crontab`  | render                          |
| `bin/executable_cronrun`                                  | `~/bin/cronrun`                  | cronrun                         |
| `bin/executable_cronctl`                                  | `~/bin/cronctl`                  | sync (core), remote (fleet)     |
| `.chezmoiscripts/run_onchange_after_sync_crontab.sh.tmpl` | (script)                         | sync                            |
| `sase/skills/crontab.md` (+ generated copies)             | global `/crontab` skill          | agent-docs                      |
| `sase/memory/crontab.md`                                  | global reference memory          | agent-docs                      |

> [!decision] cutover = supervised Drop `retire.crontab.tmpl`, the `retire:` key, and
> every retire step below. The `migrate` phase then performs a supervised manual cutover
> instead (see that phase).

### Data schema (`cron:` in `home/.chezmoidata.yaml`)

```yaml
cron:
  version: 1
  # Canonical machine name (= SSH alias in tailnet.conf) -> hostnames that
  # .chezmoi.hostname may report. Listing a machine enrolls it; an unlisted
  # machine is never touched. This map holds the only machine-specific facts.
  machines:
    athena: [athena]
    apollo: [apollo]
    mac: [Kellys-MacBook-Pro]
  # Written at the top of every managed block, sorted by key (template maps
  # iterate sorted). Values must be machine-agnostic.
  env:
    MAILTO: bryanbugyi34@gmail.com
  jobs:
    - name: example-job # ^[a-z0-9][a-z0-9-]*$, unique
      machines: [athena] # canonical names, or [all]
      schedule: "0 * * * *" # five fields or an @keyword
      description: One line # optional
      enabled: true # optional, default true; false renders a commented line
      command: some-tool --flag # one line, run by cronrun under bash
  # TEMPORARY migration aid: exact legacy lines that sync removes from outside
  # the block on that machine, in the same install. Delete once status shows
  # none remain.
  retire:
    athena: []
```

Validation runs in the partials via `fail`, so a bad file aborts `chezmoi apply` before
any crontab changes. `cronctl check` runs the same validation. It must reject all of
these:

- a `version` other than 1, or an empty `machines` map;
- a machine name not matching `^[a-z0-9][a-z0-9-]*$`, or one hostname claimed by two
  machines;
- a job name that is invalid or duplicated;
- an unknown key in a job, which catches typos like `machine:`;
- empty `machines`, or an unknown machine name (`all` is allowed, but only alone);
- a `schedule` that isn't five fields of `[0-9A-Za-z*/,-]` or one of `@reboot`,
  `@yearly`, `@annually`, `@monthly`, `@weekly`, `@daily`, `@midnight`, or `@hourly`;
- a `command` that is missing, empty, or multi-line;
- a multi-line `description` or env value, or an env key that doesn't match
  `^[A-Z_][A-Z0-9_]*$`;
- a non-boolean `enabled`;
- a `retire` key that isn't a known machine;
- `/home/`, `/Users/`, or `/run/user/` inside any `command` or env value (ADJ-4). Use
  `$HOME`; cronrun derives runtime dirs. `retire` lines are exempt.

There are no per-machine overrides. Different behaviour on two machines means two jobs.
Schedules run in each machine's **local time** (athena and the Mac use New York, apollo
uses UTC). The repo is public, so job definitions must never hold secrets; they may only
reference encrypted files by path.

### Machine resolution and fail-closed rule

The canonical name is the `cron.machines` entry whose hostname list contains
`.chezmoi.hostname`. For an **unknown** machine, `managed.crontab`, `retire.crontab`,
and the hook all render empty. chezmoi then creates no target and skips the
whitespace-only script, so the crontab stays untouched. An **enrolled** machine with no
jobs still renders a block with only markers and env lines, so deleting a machine's last
job works. Nothing ever runs `crontab -r`.

### Rendered block format

```text
# BEGIN chezmoi-cron (machine: athena; source: cron: in home/.chezmoidata.yaml; manage with cronctl)
MAILTO=bryanbugyi34@gmail.com
# bob-nightly: Nightly Bob maintenance
30 3 * * * "$HOME/bin/cronrun" bob-nightly '<quoted command>'
# tman (disabled): 0 4 * * * "$HOME/bin/cronrun" tman '<quoted command>'
# END chezmoi-cron
```

- **Quoting.** The command is single-quoted for `/bin/sh`, with each `'` written as
  `'\''`, and then each `%` is written as `\%` for cron. Jobs render in data order.
- **Description.** A description, when present, renders as the comment line above its
  job.
- **Markers.** Marker detection matches lines that start with `# BEGIN chezmoi-cron` and
  lines that equal `# END chezmoi-cron`.
- **Shared env.** Cron env lines are positional, so the block's env also applies to
  unmanaged lines below the block. That is accepted and documented.

### Runtime state (`~/.local/state/cron/`)

| File                          | Purpose                                                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `<job>.log` and `<job>.log.1` | Job output, rotated past 1 MiB, one generation kept                                                                      |
| `<job>.status`                | `key=value` lines: `job`, `host`, `pid`, `start`, `end`, `exit` (UTC ISO-8601; `end` and `exit` are empty while running) |
| `<job>.lock`                  | Overlap guard                                                                                                            |
| `sync.lock`                   | Sync serializer                                                                                                          |
| `installed.block`             | The last block sync installed                                                                                            |
| `backups/crontab.<UTC stamp>` | Whole-table backups (keep 20)                                                                                            |

> [!decision] log_dir = var_tmp Add an optional job key `log:` (a bare filename,
> validated `^[A-Za-z0-9._-]+$`). When it is set, cronrun appends to `/var/tmp/<log>`
> instead of `<job>.log`, and `cronctl logs` reads that file. The `migrate` phase sets
> `log:` to each job's legacy filename.

### `cronctl` command surface

| Command                                                          | Phase  | Behaviour                                                |
| ---------------------------------------------------------------- | ------ | -------------------------------------------------------- |
| `cronctl list [--json] [--source DIR]`                           | sync   | Job × machine matrix from data (offline)                 |
| `cronctl render <machine> [--source DIR]`                        | sync   | Desired block for any machine (offline, no SSH)          |
| `cronctl check [--source DIR]`                                   | sync   | Render and validate every machine; non-zero on any error |
| `cronctl sync [--dry-run [--block FILE]]`                        | sync   | Local reconciler; the only installer                     |
| `cronctl status [<machine>…\|--all] [--json] [--all-schedulers]` | remote | Live state per machine                                   |
| `cronctl diff <machine> [--source DIR]`                          | remote | Desired vs live block, plus outside and retire lines     |
| `cronctl apply <machine>…\|--all [--rev REV]`                    | remote | Converge targets to a pushed revision                    |
| `cronctl logs <job> [--machine M] [-n N]`                        | remote | Tail a job log                                           |
| `cronctl run <job> [--machine M]`                                | remote | Run a job now via cronrun (real side effects)            |
| `cronctl edit`                                                   | remote | Human replacement for `crontab -e`                       |

`--source DIR` takes a chezmoi repo root, such as an agent's uncommitted linked
checkout. Without it, cronctl uses the local chezmoi source.

- **Exit codes:** `0` ok, `1` failure, `2` usage or validation error, and `3` when at
  least one target is unreachable (pending) and none failed.
- **Portability:** both scripts are bash **3.2-compatible**. That rules out associative
  arrays, `mapfile`, `${x,,}`, and GNU-only flags. They must not need `jq` or Python;
  chezmoi templates do all YAML parsing, using `toJson` where structure is needed. Use
  `sha256sum` or `shasum -a 256`, whichever exists. Use `flock` when present and fall
  back to a `mkdir` lock with stale-PID detection otherwise. Tests must exercise the
  fallback path on Linux, for example through a `CRONCTL_LOCK_IMPL=mkdir` override.

## Phases

### Phase render: cron data schema, partials, and per-machine rendering

Build the declarative layer in the chezmoi repo. No crontab is touched in this phase.

1. Create `home/.chezmoidata.yaml` with the `cron:` key from the shared contract:
   `version: 1`, `machines` enrolling `athena`, `apollo`, and `mac`, `env` with
   `MAILTO`, `jobs: []`, and `retire: {}`. Add a header comment explaining that only
   `machines:` holds machine-specific facts. Jobs arrive in `migrate`.
2. Add `.chezmoitemplates/` partials. Each one takes a `dict` and renders a **whole**
   output, because v2.13.1 has no `includeTemplate`:
   - `cron-validate` (data) renders nothing or calls `fail`.
   - `cron-machine` (`hostname`, `cron`) renders the canonical name or nothing.
   - `cron-block` (`machine`, `cron`) validates, then renders the block in the contract
     format.
   - `cron-retire` (`machine`, `cron`) renders that machine's retire lines, one per
     line.

   Pick the factoring that keeps each entry template a one-liner. Repeating a small
   resolver loop is acceptable when a partial can't be captured.

3. Add `dot_config/cron/managed.crontab.tmpl` and `dot_config/cron/retire.crontab.tmpl`.
   Both resolve this machine from `.chezmoi.hostname` and render empty for an unknown
   machine. `managed.crontab` always holds the marked block for an enrolled machine.
   `retire.crontab` is empty when the machine has no retire lines.
4. Add `tests/bash/cron_render_test.sh`, rendering through
   `chezmoi --config <empty> --source <repo-root> execute-template`. Cover these:
   - every enrolled machine renders from the real data;
   - golden output for a fixture YAML, put in a temp copy of the source tree;
   - escaping of `'` and `%`, with a command containing `'`, `%`, `$(date +%F)`, a pipe,
     and `$HOME`;
   - disabled jobs, `[all]`, and env lines sorted by key;
   - an unknown hostname rendering nothing for both target files;
   - an enrolled machine with zero jobs rendering markers only;
   - one `fail` case for each validation rule.

   Also feed one rendered line through `/bin/sh` (cron's `\%` unescape simulated with
   `sed`) to prove the quoting round-trips.

5. Run the tests with the latest chezmoi and, on athena, with v2.13.1
   (`~/.nix-profile/bin/chezmoi`). Both must pass. Run `just lint` and `just test`.

Done when both chezmoi versions render identical blocks, every validation rule has a
failing test, and unknown machines render nothing.

### Phase cronrun: the machine-agnostic job runner

Add `home/bin/executable_cronrun`, which takes `cronrun [--dry-run] <job> <command>`. It
follows the runtime-state contract and is bash 3.2-compatible.

- **Environment.**
  1. Start from a baseline `PATH`: `$HOME/bin`, `$HOME/.local/bin`, `$HOME/.cargo/bin`,
     `$HOME/.nix-profile/bin`, `/opt/homebrew/bin`, `/usr/local/bin`, then
     `/usr/bin:/bin:/usr/sbin:/sbin`.
  2. Source `$HOME/.profile` when present. If sourcing fails, log it and carry on.
  3. On Linux, export each of these only when it is unset and its target exists:
     - `XDG_RUNTIME_DIR=/run/user/$(id -u)`;
     - `DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus`, when that socket
       exists;
     - `DISPLAY=:0`, when `/tmp/.X11-unix/X0` exists.

  This replaces athena's hard-coded `/run/user/1000` and `/home/bryan` lines.

- **Overlap guard.** Use a per-job lock. If a previous run still holds it, log one
  "skipped" line and exit 0.
- **Execution.** Run the command with `bash -c`. Append its stdout and stderr to
  `<job>.log` between timestamped start and end lines, and rotate the log past 1 MiB.
  Write `<job>.status` atomically at start and again at end.
- **Reporting.** Stay silent on success. On a non-zero exit, print
  `cronrun: <job> failed (exit N) on <host>` and the last 40 log lines to stderr, so
  cron mails them where a mail transfer agent exists. Then exit with the job's code.
- **`--dry-run`.** Resolve the environment exactly as above, then print `PATH`, the
  runtime variables, and whether the command's first word resolves (`command -v`). Exit
  non-zero if it doesn't resolve. Never run the command. Agents use this to check a job
  without side effects.

Add `tests/bash/cronrun_test.sh` with a fake `HOME`. Cover these:

- success is silent and writes the log and status;
- failure prints the summary and tail, and propagates the exit code;
- an overlap is skipped;
- the log rotates;
- `--dry-run` never executes;
- the runtime env is set only when unset;
- the `mkdir` lock fallback works.

Run `just lint` and `just test`.

> [!decision] log_dir = var_tmp Also accept `CRONRUN_LOG=<file>` (passed by the rendered
> line when a job sets `log:`), and write to `/var/tmp/<file>`. The status file still
> lives in the state dir.

### Phase sync: local reconciler, hook, and offline cronctl commands

Create `home/bin/executable_cronctl` with subcommand dispatch, `--help` for every
subcommand, and the offline commands `list`, `render`, and `check` from the contract.
They work through `chezmoi [--source DIR] execute-template` and the render partials.
`list --json` uses `toJson`. Then add the reconciler and its hook.

**`cronctl sync [--dry-run [--block FILE]]`** runs these steps in order. `--block FILE`
previews a block rendered elsewhere, such as from a checkout, in place of
`managed.crontab`. It is refused without `--dry-run`, so `managed.crontab` stays the
only thing ever installed.

1. **Resolve.** Resolve this machine through the `cron-machine` partial. If the machine
   is unknown, or `~/.config/cron/managed.crontab` is missing, warn and exit 0.
2. **Lock.** Take `sync.lock`.
3. **Read.** Run `crontab -l`. Treat `no crontab for <user>` as an empty table. Any
   other failure, such as a permission or macOS privacy (TCC) error, exits 1 and changes
   nothing.
4. **Count markers.**
   - With zero pairs, this is enrolment: insert the block at the top.
   - With exactly one well-formed pair, replace it in place.
   - Anything else (duplicates, nesting, or half a pair) is refused with exit 1, and
     nothing changes.
5. **Build the candidate.** Remove the lines listed in `~/.config/cron/retire.crontab`
   from outside the block: a line is removed when it equals a retire line after both are
   stripped of trailing whitespace. Keep every other outside line byte for byte, in
   order. If the candidate equals the live table, exit 0 quietly.
6. **Warn on hand edits.** If the live block differs from `installed.block`, warn loudly
   that a hand edit inside the block is being overwritten.
7. **Dry run.** `--dry-run` stops here. It prints a unified diff of live vs candidate
   and the list of retire lines it would remove.
8. **Back up and pre-check.** Back up the live table (keep 20). Run a native syntax
   check only where the local `crontab` supports one without installing; probe for it,
   and don't assume.
9. **Install and verify.**
   1. Re-read the table. If it changed since step 3, rebuild once. If it changed again,
      exit 1.
   2. Install with `crontab <file>`, then read the table back and compare it with the
      candidate.
   3. On a mismatch, reinstall the backup and exit 1.
   4. On success, write `installed.block` and print a one-line summary: the machine, the
      block's line count, and how many retire lines were removed.

**Hook.** Add `.chezmoiscripts/run_onchange_after_sync_crontab.sh.tmpl`. For an enrolled
machine, it renders a short bash script ending in `exec "$HOME/bin/cronctl" sync`. The
script carries inert content so chezmoi re-runs it exactly when this machine's inputs
change:

- this machine's rendered block and retire lines, inside a quoted heredoc to `:`;
- `# cronctl sha256: {{ include "bin/executable_cronctl" | sha256sum }}`.

For an unknown machine, the hook renders whitespace only. A full `chezmoi update -a`
then runs sync after the targets are written, and chezmoi retries a failed run on the
next apply.

**Tests.** Add `tests/bash/cron_sync_test.sh` with a fake `crontab` on `PATH` and a fake
`HOME`. Cover these:

- the first install keeps foreign lines, including a watchdog-style line;
- an unchanged table is a byte-identical no-op that never calls the installer;
- a block update keeps foreign lines in place;
- "no crontab" works as an empty table;
- corrupt markers are refused;
- a permission error is not treated as empty;
- a failed verify restores the backup;
- a hand edit inside the block triggers the warning;
- retire removes only exact matches outside the block, never lines inside it;
- `--dry-run` installs nothing;
- the lock fallback works;
- the hook renders empty for an unknown hostname and contains the block for an enrolled
  one.

**Rollout safety.** Data has no jobs yet, so landing this phase installs a block that
holds only markers and `MAILTO` on each enrolled machine at its next apply. Before
finishing, preview that on athena with the checkout's script: render athena's block with
`render athena --source <checkout>` into a temp file, then run
`sync --dry-run --block <file>`. Confirm the diff only adds the block above the existing
lines and removes nothing.

Run `just lint` and `just test`, with the render tests on both chezmoi versions.

> [!decision] cutover = supervised Skip retire handling in step 5, the retire heredoc
> content, and the retire tests.

### Phase remote: fleet commands for status, diff, apply, logs, run, and edit

Extend `cronctl` with the fleet commands in the contract.

- **Remote execution, no bootstrap.** For a target that isn't the local canonical
  machine, run
  `ssh -o BatchMode=yes -o ConnectTimeout=8 <alias> bash -s -- <internal-subcommand> <args>`
  with **the caller's own cronctl script on stdin**, read from `${BASH_SOURCE[0]}`. A
  target therefore needs only chezmoi and its source checkout, not a prior full update
  or `~/bin/cronctl`. For the local machine, run the same internal routine with no SSH.
  An unreachable target (SSH exit 255 or a timeout) is `pending`, never success.
  `bash -s` reads the script from stdin as it runs, so define everything in functions
  and make `main "$@"` the last line. Run every child command of the internal routines
  with `</dev/null`, so `git` or `chezmoi` can't consume the rest of the script. Add a
  test that would fail if a child read stdin.
- **chezmoi discovery on the target.** Non-interactive `PATH` doesn't include chezmoi on
  apollo or the Mac. Check `command -v chezmoi`, then `~/.nix-profile/bin`,
  `/opt/homebrew/bin`, `/usr/local/bin`, `~/bin`, `~/go/bin`, and `~/.local/bin`.
- **`apply`**:
  1. Determine REV. By default, take the remote head of the local chezmoi source's
     default branch (`git ls-remote`). With `--rev`, refuse unless REV is reachable from
     `origin`, so only pushed commits apply.
  2. On each target, the internal routine:
     1. runs `chezmoi git -- pull --ff-only`;
     2. verifies that REV is an ancestor of `HEAD`;
     3. checks identity: the target's hostname must resolve, through its own data, to
        the requested alias, or the target is reported as `identity_mismatch` and
        skipped;
     4. runs `mkdir -p ~/.config/cron ~/bin`, because v2.13.1's targeted apply fails
        when parent dirs are missing;
     5. runs a **targeted**
        `chezmoi apply --force ~/.config/cron ~/bin/cronctl ~/bin/cronrun`;
     6. runs the freshly deployed `~/bin/cronctl sync` **explicitly**, because a
        targeted apply runs no hooks.

     The routine leaves unrelated dotfile changes for the machine's next
     `chezmoi update -a`.

  3. Print a per-machine summary: `in_sync`, `pending`, `failed`, or
     `identity_mismatch`, with the revision. When a Mac `crontab` install fails with
     "Operation not permitted", suggest enabling Full Disk Access for remote users
     (Sharing → Remote Login) or running `chezmoi update -a` locally.

- **`status`**: for each machine, report:
  - reachability, hostname, and timezone;
  - the source `HEAD` vs `origin`;
  - the state: `in_sync`, `drifted` (the live block differs from the deployed
    `managed.crontab`), `pending` (unreachable, or the source is behind `origin`),
    `unmanaged` (enrolled, but no block yet), `identity_mismatch`, or `failed`;
  - the non-comment unmanaged lines outside the block;
  - the retire lines still present;
  - each job's last run from `<job>.status`.

  When no reachable enrolled machine still has a retire line, print
  `cron.retire can be deleted`. Add `--json` (hand-built JSON, no `jq`). Add
  `--all-schedulers`, which lists systemd user timers (Linux) and non-Apple LaunchAgents
  (macOS) **read-only**, so agents see the other schedulers without managing them.

- **`diff`**: unified diff of the desired block, rendered locally (or from `--source`),
  vs the live block. List the outside lines separately, and the retire lines that sync
  would remove.
- **`logs`** and **`run`**: default to the local machine when the job targets it.
  Otherwise require `--machine`. `run` executes `cronrun <job> '<command>'` exactly as
  cron would; its help text warns that it has real side effects.
- **`edit`**: open the data file in the local chezmoi source in `$EDITOR`, then run
  `check` and `diff` for the local machine. Finish by printing the commit → push →
  `chezmoi update -a` (or `cronctl apply`) reminder.

**Tests.** Add `tests/bash/cronctl_test.sh`. Use a fake `ssh` that runs `bash -s`
locally against a per-machine fake `HOME`, plus fake `crontab`, `git`, and `chezmoi` as
needed. Cover these:

- apply converges a target with no prior `~/bin/cronctl`;
- an unreachable target yields `pending` and exit 3;
- an unpushed `--rev` is refused;
- identity mismatch;
- the local target uses no SSH;
- status state classification, including retire lines still present;
- `--json` is valid JSON (check it with `python3 -m json.tool` in the test only);
- diff output;
- `logs` tails the right file.

Finally, against the real fleet, run read-only checks only: `cronctl status --all` and
`cronctl diff athena`. Expect the Mac may be `pending`. Run `just lint` and `just test`.

### Phase agent-docs: /crontab skill, memory note, README, and alias cleanup

**1. The `/crontab` skill.** Add the global skill `home/sase/skills/crontab.md`, with
frontmatter `name: crontab`, a one-line `description`, and `skill: true`. Model it on
`home/sase/skills/sase_gmail.md`. Then regenerate the per-provider copies with
`sase skill init` (as the repo's earlier "regenerate skills via sase skill init" commits
did) and commit those generated files too. The skill must teach the following.

- **Read.**
  - `cronctl status --all`, `cronctl list`, `cronctl render <m>`, `cronctl diff <m>`,
    and `cronctl logs <job> --machine <m>`.
  - Never describe live crontab contents from memory.
  - The Mac is often asleep, and `pending` means not yet applied.
- **Write.**
  1. Open the repo: `sase repo open chezmoi -r "<why>"`.
  2. Edit only the `cron:` data in `home/.chezmoidata.yaml`.
  3. Check the change: `cronctl check --source <checkout>`, then
     `cronctl diff <m> --source <checkout>` for each affected machine. Run
     `cronrun --dry-run <job> '<command>'` on the target when a command is new.
  4. Stage post-commit convergence:
     `sase monitor start ... -- <checkout>/home/bin/executable_cronctl apply <machines>`.
     The host starts it only after the commit is pushed. Use the checkout's script,
     because this machine's `~/bin/cronctl` may predate the commit. Its `--next` asks
     the follow-up to run `cronctl status` and report each machine as `in_sync`,
     `pending`, or `failed`. Before writing the exact flags, verify them against
     `/sase_monitor`.
  5. Finish through `/sase_final` with an explicit commit decision for the chezmoi repo.

  Humans use `cronctl edit`, then commit and push, then `chezmoi update -a` on each
  machine, or `cronctl apply`.

- **Rules.**
  - Never write a persistent job with `crontab -e`, `crontab -`, or `ssh … crontab`.
  - Never edit inside the `# BEGIN/END chezmoi-cron` block.
  - Ephemeral, self-removing automation (such as epic watchdogs) may add marker-tagged
    lines **below** the block, using a verified safe edit.
  - A user crontab never needs `sase sudo`.
  - No secrets; the repo is public.
  - Schedules use each machine's local time.
  - Time-of-day jobs on `mac` may be skipped while it sleeps.
  - Different per-machine behaviour means two jobs.

**2. The memory note.** Under `crontab_memory_note`, follow `/sase_memory_write` and
create `home/sase/memory/crontab.md`. Its frontmatter is `type: reference`,
`parent: AGENTS.md`, and a one-line description such as "Read before reading or changing
any machine's crontab — cron: data in chezmoi is the only source of truth; use cronctl
and the /crontab skill". Keep the body short:

- the source-of-truth rule;
- the block and ownership model;
- the `cronctl` command table;
- a pointer to the `/crontab` skill;
- the rules above.

Under `tailnet_memory_link`, add one line to `home/sase/memory/tailnet.md` linking
`[[crontab.md]]`. Republish as that skill directs.

**3. README.** Add a short `## Crontabs` section to the chezmoi `README.md` that covers
the data file, `cronctl --help`, and the human edit flow.

**4. Alias cleanup (ADJ-13).** In `home/dot_config/aliases.sh`, repoint `vcron` to
`cronctl edit`. Delete `vab`, `vdb`, `vwb`, and `vmb`. Their `~/Sync/bin/cron*` targets
aren't scheduled anywhere, and `~/Sync/bin/cron` doesn't exist on any machine. Leave the
`~/Sync` contents alone.

Run `just lint` (prettier) and `just test`.

### Phase migrate: move athena's and the Mac's jobs into the data and cut over

Dogfood the `/crontab` skill's write flow, reading the skill from the chezmoi checkout.
**Copy every legacy line exactly from a fresh `crontab -l`** (`ssh mac crontab -l` for
the Mac), never from this plan, because the tables may have changed since planning.

**1. Add athena's jobs** (machines: `[athena]`):

| Job             | Schedule     | Command                                                                   | State            |
| --------------- | ------------ | ------------------------------------------------------------------------- | ---------------- |
| `bob-nightly`   | `30 3 * * *` | `. "$HOME/.config/nvm/nvm.sh"; nvm use --silent default; bob nightly`     | enabled          |
| `pass-git-sync` | `0 * * * *`  | `pass_git_sync`                                                           | enabled          |
| `tman`          | `0 4 * * *`  | `tman --log=stdout:INFO@nocolor --log=/var/tmp/tman.log:DEBUG@json start` | `enabled: false` |
| `init-bg`       | `5 * * * *`  | `init-bg`                                                                 | `enabled: false` |

The `bob-nightly` command keeps the legacy `;` sequencing. Give each job a description
taken from its legacy comment. Leave the watchdog line alone; it is not data.

**2. Add the Mac's jobs** (machines: `[mac]`):

| Job                   | Schedule              | Command                                  |
| --------------------- | --------------------- | ---------------------------------------- |
| `bob-highlights-sync` | `0,15,30,45 * * * *`  | `maybe_bob_highlights_sync -w`           |
| `bob-projects-sync`   | `5,20,35,50 * * * *`  | `bob projects sync`                      |
| `bob-task-reconcile`  | `10,25,40,55 * * * *` | `bob task reconcile --retry-timeout 120` |

> [!decision] mac_schedule = live Use `*/15 * * * *` for all three jobs, and drop
> `--retry-timeout 120`.

**3. Fill `cron.retire`.**

- **`athena`:** the 5 legacy env lines, the legacy comment line above each migrated job,
  the two commented-out job lines, and the two active legacy job lines. Not the Debian
  boilerplate, and never the watchdog line.
- **`mac`:** its three live job lines, plus any comment lines directly tied to them.

**4. Verify before committing.**

- `cronctl check --source <checkout>`.
- `cronctl diff athena --source <checkout>` and `cronctl diff mac --source <checkout>`,
  when the Mac is reachable. The diff must show each legacy job removed exactly when its
  managed replacement appears, and the watchdog line kept.
- `cronrun --dry-run` for every enabled job on its machine, over SSH for the Mac. Don't
  `cronctl run` `bob-nightly`. Running `pass-git-sync` once by hand is acceptable.
- For the documented Mac schedule, confirm that `--retry-timeout` is still listed in
  `ssh mac '~/.cargo/bin/bob task reconcile --help'`.

**5. Update bob-cli's docs.**

- In `docs/vault-git-sync.md`, rewrite "Mac scheduled maintenance". Remove the claim
  "This crontab is not chezmoi-managed; it is installed by hand with `crontab` directly
  on the Mac", the `ssh mac crontab -l` / `crontab -` change procedure, and the
  `/var/tmp` log paths. Replace them with these points:
  - the chezmoi `cron:` data is the source;
  - inspect with `cronctl status mac`, `cronctl logs <job> --machine mac`, and
    `cronctl diff mac`;
  - change the schedule through the `/crontab` skill flow;
  - cronrun's failure-only mail policy.
- Check `docs/task-status-hooks.md` (around "the exact staggered crontab") and
  `docs/obsidian-sync-exclusions.md` (around "`bob nightly` crontab line"). Fix only
  statements this migration makes false.
- Follow bob-cli's docs memory (`sase memory read docs.md`), and run `just docs-check`
  in bob-cli.

**6. Commit and converge.**

1. Stage a post-commit monitor that runs the checkout's
   `home/bin/executable_cronctl apply athena apollo mac`. Its `--next` asks the
   follow-up to:
   1. run `cronctl status --all`;
   2. confirm athena is `in_sync`, with only the watchdog left unmanaged;
   3. confirm apollo is `in_sync` with an empty block;
   4. report the Mac as `in_sync` or `pending`;
   5. tell Bryan that the Mac converges at its next `chezmoi update -a` if it was
      pending, and that `cronctl logs bob-nightly` should be checked after 03:30.
2. Finish through `/sase_final`, committing both repos.

Record these `PROPOSED FOLLOW-UP:` notes on the phase bead:

- Delete `cron.retire` once `cronctl status --all` prints `cron.retire can be deleted`.
- Upgrade athena's chezmoi from v2.13.1, then move the data to `.chezmoidata/cron.yaml`.
- Remove apollo's duplicate `~/go/bin/chezmoi` dev build.

> [!decision] cutover = supervised With no retire feature, keep jobs out of the data
> until the cutover. Pick a moment away from :00 and 03:30. Back up athena's table,
> delete the legacy job and env lines with a verified safe edit, then commit with the
> post-commit apply monitor. Tell Bryan the Mac needs the same manual edit after its
> first `cronctl apply mac`.

## Out of scope

- systemd user timers (`bob-ref-intake`, `poseidon-cache-watch`) and the Mac's
  `com.bbugyi.bob-vault-sync` LaunchAgent stay as they are. Cron can't run every 15 s,
  and `status --all-schedulers` only lists these, read-only.
- Root crontabs, `/etc/cron.d`, and the work host in `.chezmoiignore` are not covered.
  Enrolling a new machine later is a one-line `machines:` addition.
- No automatic pull timer. Convergence stays a deliberate act: `chezmoi update -a`, a
  commit hook, or `cronctl apply`.
