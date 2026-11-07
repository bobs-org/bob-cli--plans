---
tier: tale
size: small
title:
  Fix cronctl's mkdir-lock spin and unterminated-block merge, then close epic bob-cli-ib
goal:
  cronctl sync can no longer hang chezmoi update -a or merge a crontab line into the END
  marker, fleet SSH calls give up on a dead link, and epic bob-cli-ib is closed.
proposed_by: bbugyi200.athena.bob-cli-ib.land
proposed_with: claude/opus
bead: bob-cli-ib
create_time: 2026-11-07 16:55:20
status: wip
---

- **PARENT:**
  [202611/crontab_as_code.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/crontab_as_code.md)
- **BEAD:** bob-cli-ib

# Fix cronctl's mkdir-lock spin and unterminated-block merge, then close bob-cli-ib

## Context

Epic `bob-cli-ib` (crontabs as code through chezmoi) has landed all six phases. Its land
agent confirmed the live fleet: athena and apollo are `in_sync`. The Mac is `unmanaged`
until Bryan runs a local `chezmoi update -a`, because macOS TCC blocks crontab installs
over SSH. Task bead `bob-cli-if` tracks that step, so it is **not** part of this tale.
The land agent's independent review of `cronctl` found two defects the epic caused, plus
two small robustness gaps. This tale fixes all four and then closes the epic.

All code work is in the **chezmoi** linked repo (`bbugyi200/dotfiles`). Open it with
`sase repo open chezmoi -r "<why>"`, read its `AGENTS.md`, and use the printed path as
`<checkout>` for every read and write. Paths below are relative to `<checkout>`. The
scripts must stay **bash 3.2-compatible**: no associative arrays, `mapfile`, `${x,,}`,
or GNU-only flags. They must not need `jq` or Python. The Mac runs `/bin/bash` 3.2.57
with BSD userland and has no `flock` on its non-interactive `PATH`.

## Changes (all in `home/bin/executable_cronctl` unless noted)

### 1. The `mkdir` sync lock must never spin (`cronctl_lock_mkdir`, ~line 866)

**Bug.** The loop only sleeps and counts toward `CRONCTL_LOCK_TIMEOUT` when it sees a
live holder. Two branches `continue` with no sleep and no count:

- `mkdir "${dir}"` fails and `${dir}` is not a directory. Examples: the state dir
  `~/.local/state/cron` is not writable (say, root-owned after a `sudo` run), or
  `sync.lock.d` is a regular file.
- A stale lock's `mv "${dir}" "${dir}.stale.$$"` fails.

Either way `cronctl sync` busy-loops at 100% CPU forever, ignoring the timeout. On the
Mac, `mkdir` is the default lock, so the `run_onchange_after_sync_crontab` hook would
hang `chezmoi update -a` with no message. The land agent reproduced it: a fake HOME with
a `chmod 555` state dir, `CRONCTL_LOCK_IMPL=mkdir`, and a 3 s timeout ran until
`timeout 8` killed it (rc 124). The `flock` path exits 1 at once.

**Fix.** Every iteration that does not take the lock must make progress toward a bounded
exit:

- When `mkdir` fails and `${dir}` exists but is not a directory, return `2` at once.
  `cronctl_lock` then dies with "cannot take the sync lock in …".
- When `mkdir` fails and `${dir}` does not exist, or the stale-lock `mv` fails: sleep 1,
  add to `waited`, and return `2` once `waited` reaches the timeout.
- Keep today's behaviour for a live holder: return `1`, "another cronctl sync still
  holds …".

**Narrow the liveness check** (`cronctl_pid_is_cronctl`, ~line 841). It treats any
command line matching `*bash*` as a live cronctl. A stale lock whose PID was recycled by
an unrelated bash process therefore costs the full timeout and an exit 1, and that
repeats for as long as the bash process lives. Only `cronctl sync` takes this lock. The
hook and the `__apply` routine both run it as `"$HOME/bin/cronctl" sync`, so the
holder's command line always contains `cronctl`. Match `*cronctl*` only, and update the
comment. The existing test `test_the_mkdir_lock_fallback_waits_for_a_live_holder` uses
`exec -a cronctl-lock-holder`, so it keeps passing.

### 2. A block without a final newline must not merge into the next line

**Bug.** `cronctl_read_block` (~line 677) reads with `|| [ -n "${line}" ]`, so it
accepts a block file whose last line, `# END chezmoi-cron`, has no trailing newline.
`cronctl_build_candidate` (~lines 775 and 780) then copies the block with
`cat "${block}"`, and the first line after the block is appended to the END marker. The
land agent's sandbox produced `# END chezmoi-cronMAILTO=old`. That outside line is
silently destroyed, and the read-back check passes because it compares against the same
candidate. Every later sync then refuses, citing "1 BEGIN marker and 0 END markers".
Nothing triggers it today: the partial renders a trailing newline. A future `-}}` trim
on the `cron-block` partial, or a `--block` file saved from `$(cronctl render …)`,
would.

**Fix.** Emit the block line by line, the same way outside lines are emitted:
`while IFS= read -r line || [ -n "${line}" ]; do printf '%s\n' "${line}"; done <"${block}"`.
Use it at both block-emission sites, through a small helper such as
`cronctl_emit_block`. Every installed block then ends in exactly one newline, whatever
the file's final byte. Check that no other path writes the block bytes raw: diff
previews, `installed.block`, and the hand-edit comparison against `installed.block`.
Normalise any you find the same way, so an unterminated `managed.crontab` neither
triggers a false "hand edit" warning nor makes `status` report `drifted`.

### 3. Fleet SSH must give up on a dead link

The fleet call (~line 1591) uses
`ssh -o BatchMode=yes -o ConnectTimeout=8 "${machine}" bash -s -- …`. `ConnectTimeout`
covers only the connect. If the Mac sleeps after the session opens, `status`, `diff`,
`apply`, `logs`, or `run` hangs instead of reporting `pending`. Add
`-o ServerAliveInterval=10 -o ServerAliveCountMax=3`, so a dead link makes ssh exit 255
(already classified as `pending`) within about 30 s. Update the header comment (~line
20), which quotes the exact command. The fake `ssh` in `tests/bash/cronctl_test.sh`
already skips `-o` options.

## Tests

Add to `tests/bash/cron_sync_test.sh`, in its existing style (fake HOME, fake `crontab`,
`run_cronctl`, `install_count`):

- **mkdir lock, `sync.lock.d` is a regular file:** with `CRONCTL_LOCK_IMPL=mkdir`, sync
  exits 1 promptly with "cannot take the sync lock". Wrap the run in `timeout` so a
  regression fails instead of hanging. Assert the table is unchanged and the installer
  was never called.
- **mkdir lock, state dir not writable** (`chmod 555` on the state dir, skipped when
  running as root): with `CRONCTL_LOCK_TIMEOUT=2`, sync exits non-zero within the
  timeout plus a small margin, with nothing installed. Restore the permissions in
  teardown.
- **Recycled PID:** a stale `sync.lock.d/pid` that names a live process whose command
  line contains `bash` but not `cronctl` is treated as stale and broken, and sync
  installs normally. Use a holder bash cannot exec-optimise, such as
  `bash -c 'sleep 30; :' &`, and kill it in teardown.
- **Unterminated block:** a `managed.crontab` written with `printf '%s'` (no final
  newline) above a table with foreign lines installs a table in which the line after
  `# END chezmoi-cron` is intact and byte-identical. A second `sync` is a no-op (zero
  further installs) with no hand-edit warning. Cover both the first install (no markers)
  and the replace-in-place path.

Add to `tests/bash/cronctl_test.sh`: one assertion that the fleet ssh invocation carries
`ServerAliveInterval`. Have the fake `ssh` log its argv to a file, if it doesn't
already.

## Verification

In `<checkout>`:

1. Run all five cron suites with athena's chezmoi v2.13.1 (the `PATH` default on athena,
   `~/.nix-profile/bin/chezmoi`):
   `bashunit tests/bash/cron_render_test.sh tests/bash/cronrun_test.sh tests/bash/cron_sync_test.sh tests/bash/cronctl_offline_test.sh tests/bash/cronctl_test.sh`.
   Run them again with a current chezmoi first on `PATH` (download the latest release
   into a temp dir if none is installed). All must pass on both.
2. `shellcheck home/bin/executable_cronctl` and
   `shfmt -i 2 -d home/bin/executable_cronctl` must be clean. `just lint` must pass.
3. `just test-bash`: the only acceptable failures are the 2 known
   `poseidon_chezmoi_isolation_test` cases, tracked by task bead `bob-cli-id` and
   failing on master since 9430387a. `test-nvim` and `test-hammerspoon` may fail locally
   on athena with "busted.runner not found", tracked by `bob-cli-ie`. Say so if they do.
4. Optional but preferred: run `tests/bash/cron_sync_test.sh` on the Mac under
   `/bin/bash` 3.2 if it is reachable
   (`ssh -o BatchMode=yes -o ConnectTimeout=8 mac true`). Skip it if the Mac is asleep
   and say so.
5. Read-only fleet check with the checkout's script:
   `<checkout>/home/bin/executable_cronctl status --all`. Expect athena and apollo
   `in_sync`, and mac `unmanaged` or `pending` (bob-cli-if). Do **not** run
   `cronctl apply`, `cronctl run`, or a non-dry-run `cronctl sync` against the real
   fleet. Never hand-edit any crontab.

The `cronctl` script's sha256 is embedded in the `run_onchange_after_sync_crontab` hook,
so each machine reruns `cronctl sync` at its next `chezmoi update -a`. That rerun is a
no-op for the table on in-sync machines, and nothing else needs deploying. The chezmoi
repo's rule is to run `chezmoi update -a --force` after a commit. Follow it by staging a
post-commit monitor through `/sase_monitor`, e.g.
`sase monitor start --reason 'Deploy the fixed cronctl on athena' --timeout 15m -- chezmoi update -a --force`.
Do not wait for that monitor, and do not order the epic closeout after it.

## Closeout of epic bob-cli-ib (final step, same turn as the code)

Do this after the verification above passes and before finishing through `/sase_final`.

1. Run `sase bead epic-symbols bob-cli-ib`. The land agent found no entries. If any
   appear, resolve each (wire it up, privatize it, or delete it per the Symvision
   epic-whitelist policy). Re-key one to a still-open bead only when that bead genuinely
   needs the exemption.
2. Close the epic. Do not use `--force`; every phase is already closed. Run
   `sase bead close bob-cli-ib --note "<verification>"` with a note covering:
   - **This tale:** the mkdir-lock spin fix, the narrowed liveness check, normalised
     block emission, and the ServerAlive options, with their tests. Record the suite
     results on chezmoi v2.13.1 and the latest, the shellcheck, shfmt, and lint results,
     and the `status --all` result.
   - **The land agent's verification:** all six phases' notes were addressed and the
     DECISIONS honored. cutover=retire is live: athena's 12 legacy lines were retired
     and the watchdog kept. mac_schedule=documented means a :00/:05/:10 stagger with
     `--retry-timeout 120`. log_dir=state: logs are in `~/.local/state/cron`.
     crontab_memory_note and tailnet_memory_link both landed.
   - **Live state and tests:** athena and apollo are `in_sync`. mac is pending its local
     apply (`bob-cli-if`). 132/132 cron tests passed on v2.13.1 before this tale.
     chezmoi CI's only failures are the poseidon tests (`bob-cli-id`).
   - **Integration:** `bob-cli-30` was closed as superseded. `bob-cli-5g` got a note to
     use the `cron:` data. bob-cli `docs/vault-git-sync.md` gained the TCC caveat and
     the nightly pointer.
   - **Follow-up triage:** `bob-cli-id`, `bob-cli-ie`, `bob-cli-if`, `bob-cli-ig`, and
     `bob-cli-ih` were filed, and the watchdog env issue was recorded on `sase-1r2`
     (details in the epic's LAND TRIAGE note).
3. Run `just symvision` in the bob-cli workspace if the recipe exists. bob-cli currently
   has none; say so if it is still missing.
4. Set `status: done` in the YAML frontmatter of the epic's plan file,
   `plan:202611/crontab_as_code.md`. That is the PLAN path `sase bead read bob-cli-ib`
   prints; resolve it there, not from memory. It currently reads `status: wip`. Change
   only that field.
5. Epic `bob-cli-ib` has no `parent_bead`, so there is no parent to close or recheck.
6. Finish through `/sase_final`, committing the chezmoi checkout (code and tests) and
   the plan-file status change.
