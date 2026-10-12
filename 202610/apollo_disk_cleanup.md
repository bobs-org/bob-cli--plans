---
tier: tale
title: Free disk space on apollo
goal:
  apollo's root filesystem regains roughly 70G by deleting only regenerable, stale, or
  duplicate files.
size: medium
decisions:
  sase_core_target:
    ask:
      Should the whole sase-core target/ build tree be deleted, not just its stale debug
      and release profiles?
    choices:
      all:
        Delete all of target/ (~22G); the next sase-core dev-update rebuilds once from
        scratch
      stale:
        Delete only target/debug and target/release (~5G); keep the warm 16G dev-update
        caches
    default: all
    why:
      Everything under target/ is rebuildable; the 16G dev-update caches are mostly
      stale artifacts
    answer: all
  old_swift:
    ask: Delete the three superseded Swift 6.1.2 toolchain copies (~7.8G)?
    default: true
    why: swiftly reports 6.4.0 in use and nothing on apollo references the 6.1.2 copies
    answer: true
proposed_by: bbugyi200.athena.0zu
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-12 14:07:12
status: wip
---

# Free Disk Space On apollo

## Goal

apollo (the DigitalOcean tailnet server, `ssh apollo`, user `bryan`) has a full root
filesystem: `/dev/vda1` showed 185G used of 193G (96%, 7.9G free) on 2026-10-12, and it
is still growing by a few GB per hour while SASE agents run there. Reclaim roughly 70G
by deleting only regenerable build output, stale agent scratch, and duplicate toolchain
copies. Do not touch anything that is user data, SASE state, or in use.

This is operations work on a remote host. **No repository files change.** Every command
below runs on apollo as `bryan`. If `hostname` already prints `apollo`, run the commands
locally; otherwise wrap them in `ssh apollo '...'`, or pipe a script with
`ssh apollo 'bash -s' < script.sh`. apollo's login shell is zsh, so run anything that
uses globs or `[[` through `bash`. Non-interactive SSH does **not** put `~/.local/bin`
on `PATH`, so call `~/.local/bin/sase` / `~/.local/bin/uv` explicitly, or
`export PATH="$HOME/.local/bin:$PATH"` first. Never use `sudo`. Keep any helper scripts
in a scratch directory outside every repository checkout.

## Findings (measured 2026-10-12, re-measure before acting)

| Candidate                                                                       |  Size | Why it is safe                                                                                                                                  |
| ------------------------------------------------------------------------------- | ----: | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `~/.cache/sase/tmp/cargo-targets/<launch>` (36 dirs)                            |   31G | Per-agent `CARGO_TARGET_DIR` + `build/`; only 8 are referenced by live processes, and those 8 are empty (4K)                                    |
| `~/projects/github/sase-org/sase-core/target`                                   |   22G | Cargo output only. The installed `sase_core_rs` is an editable install whose `.so` is a real file in `crates/sase_core_py/python/sase_core_rs/` |
| SASE owner passes (`sase disk reap --apply`)                                    |   ~9G | SASE's own reapers: 4.8G stale agent-tmp/gh-diffs scratch, plus 4.0G git-object compaction of `sase` workspace clones                           |
| `/tmp` vault-copy and benchmark scratch from 2026-10-09                         |  7.2G | bryan-owned agent scratch (copies of the Bob vault plus perf data); untouched for 3 days; `/tmp` is wiped at reboot anyway                      |
| Swift 6.1.2 copies (`~/swift-toolchain`, `~/.cache/swift-tc`, `~/.local/swift`) |  7.8G | Three copies of one old toolchain; swiftly's `config.json` has `"inUse": "6.4.0"`; no shell rc, repo, or memory references them                 |
| `~/.cache/uv` (`uv cache prune`)                                                | ≤8.9G | Removes only unreachable cache objects. Tool venvs hardlink into the cache, so the real gain is unknown                                         |

Why SASE has not reclaimed this itself: the managed-tmp reaper's dry run reports
`skipped=160, incomplete=146` for the cargo-targets. Its procfs liveness observer cannot
read `/proc/<pid>/environ` for bryan's non-dumpable daemons (`sshd`, `ssh-agent`,
`gpg-agent`, `(sd-pam)`, plus defunct zombies), so it marks every launch as "liveness
incomplete" and keeps it. `sase disk list` also runs out of scan budget and shows 0 B
for the 31G cargo-targets tree. Over non-interactive SSH, `sase disk reap`'s
`workspace_compact` fails with `FileNotFoundError: 'sase'` unless `~/.local/bin` is on
`PATH`.

## Steps

### 1. Baseline and safety checks

```bash
ssh apollo 'bash -c "hostname; df -h /; pgrep -u bryan -a -x \"cargo|rustc|maturin|clippy-driver|cargo-clippy|swift|swift-frontend|swift-build\" || echo no-build-procs"'
```

Record `df -h /` output for the final report. If any `cargo`/`rustc`/`maturin`/`swift*`
process is running, steps 3, 4 and 6 must re-check before deleting anything that process
could be using (see each step's guard).

### 2. Run SASE's own owner cleanup passes

```bash
ssh apollo 'bash -c "export PATH=\$HOME/.local/bin:\$PATH; cd ~; sase disk reap"'          # preview
ssh apollo 'bash -c "export PATH=\$HOME/.local/bin:\$PATH; cd ~; sase disk reap --apply"'  # apply
```

The preview should list `managed_tmp_reaper` (~4.8 GiB), `tool_run_retention`,
`artifact_run_retention`, and three `workspace_compact` rows (~4.1 GiB total) with
**no** `error` rows. If a `workspace_compact` row errors, report it and continue: those
passes are SASE-owned and liveness-aware. Compaction only repoints workspace `.git`
object stores at the primary's objects through alternates, repacks, and runs fsck. It
never touches working trees.

### 3. Delete dead per-agent cargo-targets

Write this script to a scratch file outside any repo. Run it once as a dry run, review
the KEEP/DELETE list, then rerun with `APPLY=1`.

```bash
#!/usr/bin/env bash
# Dry run unless APPLY=1. Run on apollo as bryan.
set -euo pipefail
shopt -s nullglob
root="$HOME/.cache/sase/tmp/cargo-targets"
me=$(id -un)

# Guard: every bryan process whose environ we cannot read must be a known
# non-build daemon or a zombie; otherwise liveness is unknown, so abort.
for p in /proc/[0-9]*; do
  [ "$(stat -c %U "$p" 2>/dev/null || true)" = "$me" ] || continue
  if ! cat "$p/environ" >/dev/null 2>&1; then
    comm=$(cat "$p/comm" 2>/dev/null || true)
    state=$(awk '{print $3}' "$p/stat" 2>/dev/null || true)
    case "$comm" in sshd|sshd-session|ssh-agent|gpg-agent|"(sd-pam)") continue ;; esac
    [ "$state" = Z ] && continue
    [ -e "$p" ] || continue   # exited while we looked
    echo "ABORT: cannot inspect live pid ${p#/proc/} ($comm)"; exit 1
  fi
done

# Launch keys any live process still references (CARGO_TARGET_DIR,
# CARGO_BUILD_BUILD_DIR, TMPDIR, SASE_LAUNCH_SCRATCH_KEY, cwd, open fds).
held=$(
  for p in /proc/[0-9]*; do
    tr '\0' '\n' < "$p/environ" 2>/dev/null \
      | sed -n -e 's#.*cargo-targets/\([^/:]*\).*#\1#p' -e 's#^SASE_LAUNCH_SCRATCH_KEY=##p' || true
    { readlink "$p/cwd"; ls -l "$p/fd" 2>/dev/null; } 2>/dev/null \
      | sed -n 's#.*cargo-targets/\([^/ ]*\).*#\1#p' || true
  done | sort -u
)

for d in "$root"/*/; do
  d=${d%/}; key=$(basename "$d")
  if grep -qxF -- "$key" <<<"$held"; then echo "KEEP held   $key"; continue; fi
  if [ -n "$(find "$d" -mmin -120 -print -quit 2>/dev/null)" ]; then
    echo "KEEP recent $key"; continue
  fi
  echo "DELETE $(du -sh "$d" | cut -f1) $key"
  if [ "${APPLY:-0}" = 1 ]; then rm -rf -- "$d"; fi
done
```

```bash
ssh apollo 'bash -s' < /path/to/scratch/reap_cargo_targets.sh            # dry run
ssh apollo 'APPLY=1 bash -s' < /path/to/scratch/reap_cargo_targets.sh    # apply
```

The two-hour quiet window matches SASE's own `managed_tmp.dead_launch.grace_seconds`
(7200). Expect roughly 29-31G back. If the guard aborts, report the uninspectable
process and skip this step. Do not weaken the guard.

### 4. Clean the sase-core build tree

Guards, all of which must pass:

- `pgrep -u bryan -a -x 'cargo|rustc|maturin|clippy-driver|cargo-clippy'` prints
  nothing.
- No process has a cwd under `~/projects/github/sase-org/sase-core/target`.
- `~/.local/share/uv/tools/sase/bin/python -c 'import sase_core_rs; print(sase_core_rs.__file__)'`
  prints a path under `crates/sase_core_py/python/`, not under `target/`. Record it.

> [!decision] sase_core_target = all
> `rm -rf ~/projects/github/sase-org/sase-core/target`

> [!decision] sase_core_target = stale
> `rm -rf ~/projects/github/sase-org/sase-core/target/debug ~/projects/github/sase-org/sase-core/target/release`
> Leave `target/uv-tool-py`, `target/uv-tool-lsp` and `target/maturin` alone.

Afterwards, rerun the `import sase_core_rs` check and `~/.local/bin/sase --version`;
both must still succeed. Do **not** run a sase-core rebuild or dev-update. The next one
SASE triggers rebuilds whatever it needs.

### 5. Delete stale /tmp agent scratch

Candidates are exactly these bryan-owned directories: `/tmp/cld_research`,
`/tmp/bobtime`, `/tmp/bobcopy`, `/tmp/bob5w-vault.wLwl`, `/tmp/sbfuzz`,
`/tmp/bob-test-root`, `/tmp/bob-timing`. For each one, delete it only if all of these
hold:

- It is owned by `bryan`.
- `find <dir> -mmin -2880 -print -quit` prints nothing (no content touched within 48h).
- No process has its cwd or an open fd under it.

Leave every other `/tmp` entry alone. That includes `tmux-*`/`ssh-*`/`systemd-*` sockets
and the recently touched `/tmp/cr_repair` and `/tmp/tsg_research`. Expect about 7.2G.

### 6. Remove superseded Swift 6.1.2 copies

> [!decision] old_swift Guards: `~/.local/share/swiftly/config.json` still says
> `"inUse" : "6.4.0"`;
> `pgrep -u bryan -a -x 'swift|swift-frontend|swift-build|swift-test'` prints nothing;
> `grep -rIl -e swift-toolchain -e swift-tc -e .local/swift ~/.zshrc ~/.zshenv ~/.zprofile ~/.profile ~/.bashrc ~/.config/sase ~/.local/bin ~/.local/share/chezmoi ~/.claude/projects 2>/dev/null`
> finds no configuration that points at the old copies. Ignore hits inside session
> transcripts. Then `rm -rf ~/swift-toolchain ~/.cache/swift-tc ~/.local/swift` and
> confirm `~/.local/share/swiftly/bin/swift --version` still reports 6.4.

> [!decision] old_swift = no Skip this step.

### 7. Prune the uv cache

```bash
ssh apollo 'timeout 600 ~/.local/bin/uv cache prune'
```

Never pass `--force` and never run `uv cache clean`: SASE agents use uv concurrently. If
the prune blocks on the cache lock until the timeout, report that and move on.

### 8. Verify and report

- `df -h /` again. Report the before/after numbers and the space reclaimed per step.
  Expect roughly 110G used / 80G free, varying with live agent activity.
- `~/.local/bin/sase --version` works, and
  `pgrep -af 'sase service run|sase scheduler run|sase axe routine'` still lists the
  SASE service processes that were running in step 1.
- Rerun `sase disk reap` (preview only) to confirm nothing errors.

### 9. File the reaper defect

Use `/sase_new_task` to file (or +1, if a duplicate exists) a `bug` task. The defect
lives in the `sase` project's managed-tmp reaper. Record the evidence above: the
dead-launch and pressure passes keep launch scratch forever whenever any of the user's
own processes has an unreadable `/proc/<pid>/environ` (`sshd`, `ssh-agent`, `gpg-agent`,
`(sd-pam)`, zombies). On apollo this left 31G of cargo-targets despite free space being
far below `managed_tmp.pressure.min_available_bytes` (32 GiB). Mention the two related
symptoms as well, in the same task or separate ones as `/sase_new_task` judges: the
`sase disk list` scan budget hides the 31G tree, and `workspace_compact` fails with
`FileNotFoundError: 'sase'` when `~/.local/bin` is not on `PATH`. Follow
`/sase_new_task`'s sizing guidance; `large` is the default when the root cause is not
proven in the Rust observer code.

## Explicitly Out Of Scope (do not delete)

- User data: `~/bob`, `~/org`, `~/zorg`, `~/.password-store`, `~/.gnupg`, `~/.ssh`,
  `~/tmp` (holds GPG key exports and an `old_sase` state backup).
- SASE state and owned caches: `~/.sase/**` (projects, artifacts, chats,
  `cache/rust-prebuild`, which SASE retains as its newest two sets), the
  `~/.local/state/sase/workspaces/**` working trees and their `registry.json`, and any
  workspace clone itself. Workspace clones may hold uncommitted agent work; only step
  2's SASE-owned compaction touches them.
- Agent CLI transcripts and state: `~/.claude`, `~/.codex`, `~/.grok`, `~/.gemini`,
  `~/.local/share/muse` (sessions).
- In-use toolchains and runtimes: `~/.rustup`, `~/.cargo`, swiftly's 6.4.0, nvm/node,
  `~/.cache/ms-playwright`, `~/.cache/puppeteer`.
- Anything needing root (`/var/log/journal`, `/var/cache/apt`, `/var/crash`,
  `/var/tmp`). These are small (~2G combined) and would need a reviewed `/sase_sudo`
  gate, so they are not worth it here.

## Acceptance

- apollo's `/` has at least ~60G free after the run, or the report explains which
  guarded steps were skipped and why.
- No SASE service process died, `sase --version` works, `import sase_core_rs` resolves
  to the source-tree `.so`, and swiftly's Swift 6.4 still runs (when `old_swift` is
  accepted).
- A bug task for the reaper defect exists, either newly filed or +1'd.
