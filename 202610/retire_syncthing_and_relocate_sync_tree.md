---
tier: tale
title: Retire Syncthing on athena and relocate the old ~/Sync tree
goal:
  Everything that broke when ~/Sync was moved (emanage, ~/.ssh/config, system backups,
  ~540 links) works again from a renamed, Syncthing-free location, and no Syncthing
  package, service, config, markers, or ~/Sync directory remain on athena.
size: medium
decisions:
  root:
    ask: Where should the relocated legacy tree (formerly ~/Sync) live?
    choices:
      db: "~/db: matches the existing $DB/$DBB/$DBH variables in ~/.profile"
      legacy:
        "~/legacy: names it as the pre-chezmoi legacy tree; $DB is repointed at it"
    default: db
    why:
      ~/.profile already calls this root $DB, and a ~/ sibling makes link retargets
      one-component swaps
    answer: db
  dotfiles:
    ask:
      Should the ~70 stow-style dotfile symlinks into the tree become real files in ~?
    choices:
      repoint:
        "No: keep them as symlinks retargeted into the tree (pure rename, lowest risk)"
      materialize:
        "Yes: move each linked file into ~ and rewrite its references; more invasive"
    default: repoint
    why:
      a pure rename fixes everything with one verifiable substitution; de-stowing can
      come later
    answer: repoint
  conflicts:
    ask:
      Should all ~400 Syncthing *.sync-conflict-* files under ~ (2020-2022) be deleted?
    choices:
      delete:
        "Yes: delete after writing a manifest; copies remain in hercules monthly/yearly
        backups"
      keep:
        "No: keep them (links retargeted); only markers, config, and the package are
        removed"
    default: delete
    why:
      they are Syncthing artifacts by definition, years stale, and still recoverable
      from hercules
    answer: delete
  old_tmp:
    ask:
      Should the old 33 GB ~/tmp contents (reachable only via ~/bar) move to
      ~/tmp/old_tmp?
    choices:
      nest:
        "Yes: move them to ~/tmp/old_tmp and delete ~/bar, as the Oct 9 history
        attempted"
      keep: "No: leave them inside the relocated tree and just delete the ~/bar link"
    default: nest
    why:
      zsh history shows `mv ~/old_tmp ~/tmp/` on Oct 9 and `ls ~/tmp/old_tmp` on Oct 12
    answer: nest
proposed_by: bbugyi200.athena.0zt
decided_by: reviewer
decided_via: tui
create_time: 2026-10-12 07:42:09
status: wip
---

# Retire Syncthing on athena and relocate the old ~/Sync tree

## Context (diagnosis — already verified, do not re-derive)

Bryan no longer uses Syncthing. On 2026-10-09 16:07 he ran
`mv ~/Sync ~/.old_syncthing_dir` and then `mv ~/.old_syncthing_dir ~/Sync`. Between
those two commands his zsh shell-history hook
(`SHV_SHELL_HISTORY_ROOT=~/Sync/var/logs/shell-history`) recreated
`~/Sync/var/logs/...`, so the second `mv` **nested** the tree. The result:

- `~/Sync/.old_syncthing_dir/` — the complete old Sync tree (~81 GB, nothing lost):
  `bin/` (git checkout of `bbugyi200/LocalAlias`, the legacy scripts), `home/` (old
  stow-style dotfiles checkout, incl. 33 GB `home/tmp`), `var/` (47 GB notes, projects,
  books, logs), `lib/`, `src/`, `include/`, `usr/`, `tmp/`, plus `.stfolder` and
  `.stignore`.
- `~/Sync/var/logs/shell-history/athena/2026/10.log` — 177 shell-history lines written
  since the move (they start at 2026-10-09 16:07:44, strictly after the last line of the
  old tree's copy of the same file).
- Everything that pointed into `~/Sync/...` is now dangling:
  - **539 symlinks under `~`**: 430 in `~/.local/bin` (incl. `emanage`, `emanage.py`),
    34 in `~/.config`, 21 in `~/.local/share` (incl. `emanage`, `fonts`, `zathura`),
    `~/projects/*`, `~/var/img`, `~/var/sandbox/2022/04/25`, and ~40 top-level dotfiles
    (`~/.bashrc`, `~/.taskrc`, `~/.task`, `~/.timewarrior`, `~/.git_template`,
    `~/.ipython`, ...).
  - **`~/.ssh/config`** (`-> ../Sync/home/.ssh/config`). It holds the `github.com` (port
    443 + SASE service key) and `github-bob` host blocks, so `bob-vault-sync.service` is
    currently failing every 20 s with `Could not resolve hostname github-bob`. **Fix
    this first.** Its content is byte-identical to the hercules backup copy.
  - **31 root-owned symlinks**: every `/etc/cron.{hourly,daily,weekly,monthly}/*_backup`
    and `*_jobs` script, `/etc/backup.sh`, `/etc/jobs.sh`, and 22 links under
    `/usr/local/{include,lib,src,share/zsh/site-functions}` (full list in the appendix).
    The cron scripts also hard-code `source /home/bryan/Sync/bin/cron.jobs/backup.sh`
    and `/home/bryan/Sync/bin/main/prometheus_disk_space_textfile`, so **system backups
    to /mnt/hercules/backup are not running.**
  - `~/.local/lib/python3.11/site-packages/__editable__.bbugyi_pylibs-2.0.0.pth`
    contains `/home/bryan/Sync/var/projects/pylibs`; that editable install provides the
    `bugyi` package that `emanage` imports.
  - chezmoi source: `home/dot_profile` (`export DB="$HOME"/Sync` plus `FAV_CLIP_FILE`,
    `RECENTLY_EDITED_FILES_LOG`, `RUST_SRC_PATH`, `SHV_SHELL_HISTORY_ROOT`),
    `home/dot_config/aliases.sh` (~25 `~/Sync` references),
    `home/dot_vim/ftplugin/{c,cpp}.vim`, and a commented line in
    `home/dot_vim/plugins.vim`.
- Syncthing remnants: package `syncthing` 1.30.0 (apt source
  `/etc/apt/sources.list.d/syncthing.list` + `.list.save`; signing key
  `37C84554E7E0A261E4F76E1ED26E6ED000654A3E` "Syncthing Release Management" inside the
  shared legacy keyring `/etc/apt/trusted.gpg`); `syncthing@bryan.service` enabled and
  running (`Restart=on-failure`, so SIGTERM stops it for good); a dangling
  `/etc/systemd/system/sleep.target.wants/syncthing-resume.service`;
  `/etc/ufw/applications.d/syncthing` (package conffile; ufw itself is not installed);
  `~/.config/syncthing/` (config, device keys, index db); `.stfolder`/`.stignore` at the
  tree root; ~400 `*.sync-conflict-*` files.
- Backups: the latest `home/hourly` snapshot is
  `/mnt/hercules/backup/home/hourly/bryan/Sync/` (2026-10-08 21:45). It is the fallback
  source for any file that turns out to be missing. It excludes `.git/` dirs and caches.
  Everything lives on one filesystem (`/`), so every `mv` below is an instant rename.

Non-root work runs directly. **Never run raw `sudo`.** All privileged steps go into
**one** batched `/sase_sudo` request (Step 10), and the successor agent finishes the job
(Step 11).

Throughout this plan `<root>` means `~/db` (`/home/bryan/db`), or `~/legacy` if decision
`root = legacy`. Both are direct children of `~`, so retargeting a link means replacing
its single `Sync` path component with the root's basename.

## Non-goals

- Dissolving the legacy tree into chezmoi, or pruning dead X11-era configs (xmonad,
  bspwm, termite, ...). Repointing keeps them exactly as functional as they were.
- Committing or pushing inside the legacy checkouts under `<root>` (`bin` = LocalAlias,
  `home` = old dotfiles clone, `lib/python`, `var/projects/*`). They are live machine
  state here. Leave path rewrites as uncommitted working-tree edits.
- Unrelated "sync" tooling must not be touched: `bob-vault-sync.service`,
  `ob-sync-bob.service`, `pass_git_sync` (its "No usable SSH agent" cron errors predate
  the move), vdirsyncer, rclone, rsync, `bob ... sync`.
- Syncthing on other machines.
- Fixing the pre-existing intermittent `/home` backup failure (see Step 11). Report it
  and file it; don't fix it.

## Steps

### 0. Capture state (read-only)

1. `mkdir -p ~/tmp/sync-removal`. All manifests below go there.
2. Confirm `~/Sync` contains only `.old_syncthing_dir/` and `var/logs/shell-history/...`
   (`find ~/Sync -mindepth 1 -maxdepth 2`). If it holds anything else, stop and report
   instead of guessing.
3. Confirm `<root>` does not exist. If decision `old_tmp = nest`, also confirm
   `~/tmp/old_tmp` does not exist.
4. Write `links-before.tsv`: every symlink under `~` whose target has a `Sync` path
   component (`find ~ -xdev` pruning `~/Sync`, `~/.local/state/sase/workspaces` and
   `node_modules`; print `%p\t%l`). Expect ~539 rows.

### 1. Stop Syncthing (no root needed)

`kill -TERM "$(systemctl show -p MainPID --value syncthing@bryan.service)"`. Then
confirm `pgrep -u bryan -x syncthing` is empty. A clean exit is not restarted under
`Restart=on-failure`. Step 10 disables and purges it.

### 2. Restore `~/.ssh/config` as a real file (both `dotfiles` branches)

1. `rm ~/.ssh/config` (it is the dangling symlink; check with `test -L` first).
2. `mv ~/Sync/.old_syncthing_dir/home/.ssh/config ~/.ssh/config && chmod 600 ~/.ssh/config`.
   It becomes a regular file, which matches the tailnet convention: chezmoi's
   `run_onchange_after_ensure_ssh_tailnet_include` script keeps `Include tailnet.conf`
   at the top of a machine-local `~/.ssh/config`. Keeping ssh config out of the 755
   legacy tree is also safer.
3. Verify: `ssh -G github-bob` shows `hostname ssh.github.com`, `port 443`, and
   `identityfile ~/.ssh/id_bob_vault`. `ssh -G apollo` resolves through `tailnet.conf`.
   About a minute later, `journalctl --user -u bob-vault-sync --since -2min` has no
   `Could not resolve hostname github-bob`.

### 3. Relocate the tree and drop `~/Sync`

1. `mv ~/Sync/.old_syncthing_dir <root>`.
2. Merge the post-move shell history. Append every file under `~/Sync/var/` to the same
   relative path under `<root>/var/` (create directories as needed). Today that is only
   `logs/shell-history/athena/2026/10.log`: append its 177 lines to the 553-line copy.
3. `find ~/Sync -type f` must be empty. Then `rm -r ~/Sync`.

Interactive shells that are still running keep the old `SHV_SHELL_HISTORY_ROOT` and will
recreate `~/Sync/var/logs/...`. Step 11 sweeps for this again.

### 4. Retarget symlinks under `~`

For each row in `links-before.tsv`, except the ones handled elsewhere (`~/.ssh/config`
in Step 2, `~/bar` in Step 5, and `~/tags.sync-conflict-*` when `conflicts = delete`):

- Replace the single `Sync` path component of the target with the root's basename and
  keep the link's relative or absolute style (`../Sync/home/.bashrc` →
  `../db/home/.bashrc`). Apply with `ln -sfn <new> <link>`; `-n` matters for links to
  directories. Record `link\told\tnew` in `links-retargeted.tsv`.
- Every retargeted link must resolve (`test -e`). Report any that do not.
- Delete these five links instead. They were already dangling before the move:
  `~/.local/bin/gcl`, `~/.local/bin/.mypy_cache/3.8`,
  `~/.local/bin/.mypy_cache/CACHEDIR.TAG`,
  `~/.local/bin/__pycache__/poll_weather.cpython-39.pyc`,
  `~/.local/share/qutebrowser/athena-sessions/.lshrc`.
- Do not touch symlinks **inside** `<root>`. The ~540 links in
  `home/tmp/dot_local_bin-2022-06-08/` were already broken before the move.

> [!decision] dotfiles = materialize Instead of retargeting a _stow-shaped_ link, where
> `~/<p>` points at `<root>/home/<p>` for the same `<p>` (for example `~/.bashrc`,
> `~/.config/htop`, `~/.local/share/fonts`, `~/var/img`), `rm` the link and
> `mv <root>/home/<p> ~/<p>`. Retarget every other link as above (`~/.local/bin/*`,
> `~/projects/*`). In Step 6, before the generic substitution, rewrite references to
> `…/Sync/home/<p>` and `$DBH/<p>` for each materialized `<p>` so they point at the same
> prefix style plus `/<p>` (`/home/bryan/Sync/home/.timewarrior` →
> `/home/bryan/.timewarrior`), including references inside the materialized files
> themselves. Write `materialized.txt`. Leave the rest of `<root>/home` (`.git`,
> `projects`, `var`, ...) in place.

### 5. Old `~/tmp` contents

`~/bar` is the original 2020 `~/tmp -> Sync/home/tmp` symlink, parked there on Oct 9.
The current `~/tmp` is a fresh real directory. Check `test -L ~/bar`, then `rm ~/bar`.

> [!decision] old_tmp = nest `mv <root>/home/tmp ~/tmp/old_tmp`. Exclude `~/tmp/old_tmp`
> from every `Sync` sweep below; its old broken link farm is user scratch.

With `old_tmp = keep`, leave `<root>/home/tmp` where it is.

### 6. Rewrite hard-coded `~/Sync` paths

Rewrite occurrences of the old root written as `/home/bryan/Sync`, `~/Sync`,
`$HOME/Sync`, `${HOME}/Sync`, `"$HOME"/Sync`, `"${HOME}"/Sync`, or
`os.environ['HOME']}/Sync`, followed by `/`, a quote, whitespace, or end of line, to the
same prefix plus `/<basename>`. For example:

```
sed -E -i 's#(/home/bryan|~|\$HOME|\$\{HOME\}|\$HOME"|\$\{HOME\}"|HOME'"'"'\]\})/Sync([/"'"'"' ]|$)#\1/db\2#g'
```

(Use `legacy` instead of `db` in the replacement under `root = legacy`.)

After that, re-grep for remaining `Sync` path references and fix them by hand. Apply
only to text files (`grep -I`), and write the touched paths to `rewritten-files.txt`.

In scope:

- Inside `<root>`: `bin/`, `lib/`, `src/`, `include/`, `usr/`, and `home/`. Skip
  `home/.git`, `home/tmp`, `home/projects`, and `home/var`. Everywhere, skip `.git/`,
  `.mypy_cache/`, `__pycache__/`, `.pytest_cache/`, and `*.sync-conflict-*`. This covers
  the cron.jobs scripts (`backup.sh` sourcing, `hourly_jobs`' collector path, comments),
  ~50 scripts in `bin/main`, `bin/zathura/zopen.py`, `bin/xmonad/*`, and the zopen,
  zathura and okular state under `home/.local/share` (book paths).
- Outside `<root>`: the unmanaged `~/.config/tmuxinator/*.yml` files that mention
  `Sync`, `~/.config/QtProject.conf`, `~/.config/qutebrowser/qsettings/QtProject.conf`,
  `~/.config/xfce4/desktop/icons.screen0*`, and `~/.config/calibre/*.json` (only if
  calibre is not running).
- Python path files:
  `~/.local/lib/python3.11/site-packages/__editable__.bbugyi_pylibs-2.0.0.pth` (one line
  becoming `/home/bryan/db/var/projects/pylibs`), and
  `~/.pyenv/versions/3.7.12/lib/python3.7/site-packages/{bbugyi-pylibs.egg-link,easy-install.pth}`.

Never rewrite:

- `<root>/var/**`. Its only `Sync` hits are mypy caches and historical shell-history
  logs.
- Shell histories, or anything under `~/.sase/`.
- `~/.config/aliases.sh` directly. It is chezmoi-rendered, so Step 7 edits its source.

### 7. chezmoi source

1. `sase repo open chezmoi -r "Repoint ~/Sync references after retiring Syncthing"`,
   then read the opened repo's `AGENTS.md`.
2. Edit `home/dot_profile`: `export DB="$HOME"/db` (`DBB` and `DBH` derive from it),
   plus `FAV_CLIP_FILE`, `RECENTLY_EDITED_FILES_LOG`, `RUST_SRC_PATH`, and
   `SHV_SHELL_HISTORY_ROOT`.
3. Edit `home/dot_config/aliases.sh`, `home/dot_vim/ftplugin/c.vim`,
   `home/dot_vim/ftplugin/cpp.vim`, and the commented line in
   `home/dot_vim/plugins.vim`. Afterwards `grep -rnw Sync home` should hit only
   unrelated prose, such as `pass_git_sync`'s "Sync the pass password store".
4. Apply now, scoped to these targets:
   `chezmoi apply --force --source <opened chezmoi path> ~/.profile ~/.config/aliases.sh ~/.vim/ftplugin/c.vim ~/.vim/ftplugin/cpp.vim ~/.vim/plugins.vim`.
   chezmoi honors `.chezmoiroot`. Check that
   `bash -c '. ~/.profile; echo "$DB $SHV_SHELL_HISTORY_ROOT"'` prints the new paths.
5. The commit lands through the final declaration (chezmoi becomes a repo obligation).
   chezmoi's AGENTS.md requires `chezmoi update -a --force` after commits, and agents
   may not mutate after declaring, so the final report must tell Bryan to run it once
   the commit lands.

### 8. User-level Syncthing remnants

1. `rm -r ~/.config/syncthing`.
2. `rm -r <root>/.stfolder` and `rm <root>/.stignore`. Delete any `.stversions`,
   `.syncthing.*.tmp`, or `~syncthing~*.tmp` found under `~` (none were seen).
3. Conflicts. With `conflicts = delete`: write `find ~ -xdev -name '*.sync-conflict-*'`
   (pruning `~/.local/state/sase/workspaces`) to `conflicts-deleted.txt`, then delete
   those paths. This includes symlinks such as
   `~/tags.sync-conflict-20210729-012908-PMZBLBJ` and both the `Trash/files` and
   `Trash/info` entries. With `keep`, do nothing here.
4. `fc-cache -f` so the retargeted `~/.local/share/fonts` is picked up.

### 9. Non-root verification (before asking for privilege)

- `command -v emanage` resolves to `~/.local/bin/emanage`, and `emanage -h` exits 0. Do
  **not** run `-u`/`-c`.
- `python3 -c 'import bugyi; print(bugyi.__file__)'` prints a path under
  `<root>/var/projects/pylibs`.
- `bash -n` passes for every script in `<root>/bin/cron.jobs/**`, and
  `grep -rn 'Sync' <root>/bin/cron.jobs` finds nothing.
- `git config --get init.templatedir` (if set) names a directory that exists.
- No symlink under `~` has a `Sync` path component in its target (pruning `<root>`,
  `~/tmp/old_tmp`, and `~/.local/state/sase/workspaces`). `test -e ~/Sync` fails, or
  only the shell-history path has reappeared (see Step 11).
- The Step 2 ssh checks still pass.

### 10. One batched privileged request via `/sase_sudo`

Use the `/sase_sudo` skill. Submit **one** request with `stop_on_failure: true` and
`output_to_agent: "tail"`. Each command gets an id, exact argv, and a one-line why.
Order the commands so that the critical backup fix runs first:

1. One `ln -sfn <new-target> <link>` per row of the appendix: 31 commands, each target's
   `Sync` component swapped for the root basename, relative style preserved (e.g.
   `ln -sfn ../../home/bryan/db/bin/cron.jobs/cron.hourly/hourly_backup /etc/cron.hourly/hourly_backup`).
2. `run-parts --test /etc/cron.hourly`, then the same for `cron.daily`, `cron.weekly`,
   and `cron.monthly`. The `*_backup` and `*_jobs` scripts must be listed.
3. `systemctl disable --now syncthing@bryan.service`.
4. `rm -f /etc/systemd/system/sleep.target.wants/syncthing-resume.service`, then
   `systemctl daemon-reload`.
5. `apt-get purge -y syncthing`. This also removes the unit template and
   `/etc/ufw/applications.d/syncthing`.
6. `rm -f /etc/apt/sources.list.d/syncthing.list /etc/apt/sources.list.d/syncthing.list.save`.
7. `gpg --no-default-keyring --keyring /etc/apt/trusted.gpg --batch --yes --delete-keys 37C84554E7E0A261E4F76E1ED26E6ED000654A3E`.
   Delete only this fingerprint: the keyring also holds the Google, Plex, vdirsyncer,
   Docker, and VirtualBox keys.
8. `apt-get update` (`timeout_seconds` 600).
9. Diagnostics, last because grep exits 1 when nothing matches:
   `find /root -xdev -maxdepth 4 -lname '*Sync*' -printf '%p -> %l\n'` and
   `grep -rn Sync /var/spool/cron/crontabs`.

Set `next.prompt` to: read the sudo ledger, then run Step 11 of this plan.

### 11. Successor: verify privileged results and wrap up

1. Ledger: every command succeeded, apart from diagnostic grep exits of 1.
   `dpkg -s syncthing` reports not installed. `systemctl list-unit-files 'syncthing*'`
   is empty. No `syncthing` entry remains in `/etc/apt/sources.list.d`, and no Syncthing
   key is left in `/etc/apt/trusted.gpg`. `find /etc /usr/local -xdev -lname '*Sync*'`
   is empty, and every retargeted system link resolves.
2. If the diagnostics found `/root` links or root crontab lines pointing into `Sync`,
   submit a second `/sase_sudo` request that retargets them by the same rule.
3. Backups. Use `/sase_monitor` to poll `/mnt/hercules/backup/home/hourly/backup.txt`
   until its epoch is newer than the time the cron links were fixed (timeout about 2 h,
   covering the next `:17` hourly run plus the rsync).
   - If it lands, report the timestamp.
   - If it does not, report that clearly. Snapshots from before the move already showed
     `/home` failures: `home/hourly` was only refreshed at Oct 7 22:38, Oct 8 04:35,
     05:33 and 21:45, and `home/daily` was skipped on Oct 8 and Oct 9 while the
     `etc`/`usr`/`var` dailies succeeded on Oct 9. Treat it as pre-existing and out of
     scope, and file a bug task bead with that evidence via `/sase_new_task`.
4. Final sweep. If `~/Sync` has reappeared, which happens when one of Bryan's old shells
   ran a command, append its shell-history lines to `<root>/var/...` exactly as in Step
   3 and remove it again.
5. Final report:
   - Root cause: the nested `mv`.
   - What moved, how many links were retargeted, and how many files were rewritten.
   - Where the manifests are: `~/tmp/sync-removal/`.
   - Backup status.
   - Reminders for Bryan: run `chezmoi update -a --force` once the chezmoi commit lands.
     Restart open shells and tmux (`exec zsh -l`), or old shells will recreate `~/Sync`.
   - A one-liner to merge stray history and `rm -r ~/Sync` if it reappears later.

## Rollback

Every move is a same-filesystem rename, and `links-before.tsv` and
`links-retargeted.tsv` record each link change. Missing files can be restored from
`/mnt/hercules/backup/home/hourly/bryan/Sync/` (Oct 8 21:45 snapshot).

## Appendix: root-owned symlinks to retarget (current targets)

```
/etc/backup.sh -> ../home/bryan/Sync/bin/cron.jobs/backup.sh
/etc/jobs.sh -> ../home/bryan/Sync/bin/cron.jobs/jobs.sh
/etc/cron.hourly/hourly_backup -> ../../home/bryan/Sync/bin/cron.jobs/cron.hourly/hourly_backup
/etc/cron.hourly/hourly_jobs -> ../../home/bryan/Sync/bin/cron.jobs/cron.hourly/hourly_jobs
/etc/cron.daily/daily_backup -> ../../home/bryan/Sync/bin/cron.jobs/cron.daily/daily_backup
/etc/cron.daily/daily_jobs -> ../../home/bryan/Sync/bin/cron.jobs/cron.daily/daily_jobs
/etc/cron.weekly/weekly_backup -> ../../home/bryan/Sync/bin/cron.jobs/cron.weekly/weekly_backup
/etc/cron.weekly/weekly_jobs -> ../../home/bryan/Sync/bin/cron.jobs/cron.weekly/weekly_jobs
/etc/cron.monthly/monthly_backup -> ../../home/bryan/Sync/bin/cron.jobs/cron.monthly/monthly_backup
/usr/local/include/daemonize.h -> ../../../home/bryan/Sync/include/daemonize.h
/usr/local/include/errorwraps.h -> ../../../home/bryan/Sync/include/errorwraps.h
/usr/local/include/gutils.h -> ../../../home/bryan/Sync/include/gutils.h
/usr/local/lib/libdaemonize.a -> ../../../home/bryan/Sync/lib/C/libdaemonize.a
/usr/local/lib/liberrorwraps.a -> ../../../home/bryan/Sync/lib/C/liberrorwraps.a
/usr/local/lib/python/.archive.tar.gz -> ../../../../home/bryan/Sync/lib/python/.archive.tar.gz
/usr/local/lib/python/tags -> ../../../../home/bryan/Sync/lib/python/tags
/usr/local/share/zsh/site-functions/_chezmoi -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_chezmoi
/usr/local/share/zsh/site-functions/_cim -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_cim
/usr/local/share/zsh/site-functions/_him -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_him
/usr/local/share/zsh/site-functions/_jrnl -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_jrnl
/usr/local/share/zsh/site-functions/_lim -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_lim
/usr/local/share/zsh/site-functions/_pip -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_pip
/usr/local/share/zsh/site-functions/_vlog -> ../../../../../home/bryan/Sync/usr/local/share/zsh/site-functions/_vlog
/usr/local/src/ipylint-backup -> ../../../home/bryan/Sync/src/ipylint-backup
/usr/local/src/libdaemonize -> ../../../home/bryan/Sync/src/libdaemonize
/usr/local/src/liberrorwraps -> ../../../home/bryan/Sync/src/liberrorwraps
/usr/local/src/lockfile -> ../../../home/bryan/Sync/src/lockfile
/usr/local/src/panel -> ../../../home/bryan/Sync/src/panel
/usr/local/src/switch-or-run -> ../../../home/bryan/Sync/src/switch-or-run
/usr/local/src/tqueue -> ../../../home/bryan/Sync/src/tqueue
/usr/local/src/window-stack -> ../../../home/bryan/Sync/src/window-stack
```
