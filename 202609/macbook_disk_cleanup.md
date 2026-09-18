---
tier: tale
title: Free disk space on kellys_mbp without breaking anything
goal:
  Reclaim roughly 55-60 GB on the nearly full Data volume by removing only regenerable
  build outputs and caches, owner-managed SASE and package-manager data, and
  verified-redundant copies, while every tool, repo, workspace, and user account keeps
  working.
size: medium
proposed_by: bbugyi200.kellys_mbp.0y
create_time: 2026-09-18 18:14:47
status: wip
---

# Free disk space on kellys_mbp without breaking anything

## Goal

The Data volume is 95% full: 422.7 GB used and about 23–24 GB free, measured 2026-09-18.
Reclaim about 55–60 GB by removing only three kinds of data:

1. regenerable data (build outputs and package caches);
2. data whose owning tool already knows how to reclaim it safely (`sase disk reap`,
   `brew`, `uv`, `npm`);
3. duplicate or stale copies whose redundancy has been verified.

Every step below has a guard. When a guard fails, skip that item, record the skip, and
continue. Do not improvise deletions that are not listed here.

This is machine maintenance only. It makes **no repo changes and no commits**.

## What the investigation found (2026-09-18)

- `df`: `/System/Volumes/Data` uses 394 GiB and has 23 GiB free. Preboot is 18.1 GB,
  which is inflated by a staged macOS update. `macOS Tahoe 26.7` is pending.
- The `bbugyi` user can see only about 174 GiB of the Data volume. The other ~220 GiB is
  data this account cannot read:
  - `/Users/kellydonnelly`, an active account. Its Desktop, Downloads (1,128 items), and
    Documents were modified in 2026, and its Trash holds 63 items.
  - Folders protected by macOS privacy controls (TCC): `~/.Trash`, Mail, Messages, and
    Photos.
  - System stores such as Spotlight and `AssetsV2`.

  The implementer must not touch any of it (see "Manual follow-ups").

- Largest reclaimable items in the `bbugyi` account:

  | Item                                                                                                     | Size                                    | Nature                                                                                                                                                                          |
  | -------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Git objects duplicated across the numbered `sase` workspace checkouts                                    | ~16.1 GiB (`sase disk reap` dry run)    | SASE-owned compaction                                                                                                                                                           |
  | `~/projects/github/sase-org/sase-core/target`                                                            | 13 GB                                   | Cargo build output (active repo, rebuilds on next build)                                                                                                                        |
  | Other Cargo `target/` dirs (sase-core and bob-cli workspace checkouts, bob-cli, actstat, zorg primaries) | ~10 GB                                  | Cargo build output                                                                                                                                                              |
  | `~/tmp/sase/cargo-targets`                                                                               | 2.7 GB                                  | Legacy SASE cargo target root. SASE now uses `~/.cache/sase/tmp/cargo-targets`. Nothing configured references it, and it was last written 2026-09-14                            |
  | `~/projects/github/bbugyi200/bob-mac-capture/.build`                                                     | 387 MB                                  | SwiftPM build output. The installed app runs from `~/Applications/Bob Mac Capture.app`                                                                                          |
  | `~/.cache/uv`                                                                                            | 2.0 GB                                  | uv cache                                                                                                                                                                        |
  | `~/.npm/_cacache`                                                                                        | 1.7 GB                                  | npm cache                                                                                                                                                                       |
  | Homebrew caches / old versions                                                                           | ~0.8 GB (`brew cleanup -n --prune=all`) | brew-owned                                                                                                                                                                      |
  | `~/Library/Caches/com.pinterest.PINDiskCache.PINCacheShared`                                             | 1.3 GB                                  | App cache, untouched since 2026-07-05, no open handles                                                                                                                          |
  | `~/cutover-backups/restores/kellys_mbp-20260906T172717-0aec9e-20260906T173029-42893e`                    | 5.7 GB                                  | Rehearsal restore of the SASE x7.6 cutover backup, which sits beside it                                                                                                         |
  | `~/cutover-backups/backups/kellys_mbp-20260906T172717-0aec9e`                                            | 5.8 GB                                  | The cutover backup itself (keep, but compress)                                                                                                                                  |
  | `~/var/backups/bob-pre-gitsync-macbook-20260827T125528`                                                  | 2.9 GB                                  | Vault backup taken before git sync (keep, but compress)                                                                                                                         |
  | `~/tmp/old_sase`                                                                                         | 1.1 GB                                  | Pre-migration SASE state copy from 2026-08-31 (keep, but compress)                                                                                                              |
  | `~/tmp/git@github.com:sase-org/sase--research`                                                           | 614 MB                                  | Clone in a mistaken location. Clean and fully pushed                                                                                                                            |
  | `~/projects/github/bbugyi200/bob`                                                                        | 1.6 GB                                  | Stale second clone of the Bob vault (last commit 2026-06-13). Its HEAD exists in the live vault `~/bob`, it is not registered in Obsidian, and nothing configured references it |

## Hard safety rules (read before running anything)

1. **Never run `du` bare.** The user's shell aliases `du` to `sudo ncdu`. Always call
   `/usr/bin/du`. Never use `sudo`: nothing in this plan needs root, and SASE hooks
   block raw sudo.
2. **Do not touch** any of the following:
   - other users (`/Users/kellydonnelly`, `/Users/shelter`, `/Users/Shared`) and
     `/Applications` (apps are shared with Kelly);
   - `/System`, `/Library`, `/private`, Preboot, APFS snapshots, `sleepimage`, and
     `/Library/Updates`;
   - `~/bob` (the live vault), `~/org`, `~/Downloads`, and `~/Documents`;
   - anything under `~/Library` except the single cache named in step 3;
   - `~/.sase` (managed by `sase disk reap`), `~/.cache/sase`, `~/.local/share/claude`,
     `~/.rustup`, and `~/.cargo`;
   - `~/tmp/chezmoi_build` (chezmoi's neovim build dir, reused by `chezmoi apply`);
   - every primary git checkout except the one stale vault clone in step 4c.
3. **Never run `git gc`, `git prune`, or `git repack` by hand in SASE repos or
   workspaces.** After `sase disk reap` compaction, workspaces borrow objects from a
   shared source. A manual prune could corrupt them. Leave git maintenance to SASE.
4. **Do not use `cargo clean`.** Agent shells export `CARGO_TARGET_DIR` and
   `CARGO_BUILD_BUILD_DIR`, which point at SASE's managed target dir. `cargo clean`
   would wipe that dir instead of the in-tree `target/`. Use the guarded `rm -rf` in
   step 2.
5. Quote every path. Many contain spaces (`Application Support`), and one contains `:`
   and `@`.
6. Before each deletion, re-check its guard and measure it with `/usr/bin/du -sk`.
   Record `item | size | action (deleted/archived/skipped) | reason` in a running table
   for the final report.

## Steps

### 0. Baseline and preflight

- Record the baseline with `df -k /System/Volumes/Data` and
  `diskutil info disk3s5 | grep -E 'Used|Free'`.
- Record `sase doctor 2>&1 | tail -3`. The baseline on 2026-09-18 was
  `OK: 55, WARN: 13, ERROR: 3, SKIP: 5`. The goal is no _new_ ERROR or WARN afterwards.
- Record `sase agent list`, so you know which agents are live, and `bob --version`.
- Check that `zstd` is available: `command -v zstd`. It is expected at
  `/opt/homebrew/bin/zstd`.

### 1. SASE-owned cleanup

- Run `sase disk reap` (dry run). Confirm that it still shows `workspace_compact`
  reclaiming about 16 GiB for `gh_sase-org__sase` and reports no problems.
- Run `sase disk reap --apply`.
- Then, for every numbered checkout listed by `sase workspace list -a` for project
  `sase` whose path exists, verify:
  - `git -C "<path>" rev-parse HEAD` works;
  - `git -C "<path>" status --porcelain` runs without error (content may be non-empty);
  - `git -C "<path>" fsck --connectivity-only --no-dangling` exits 0.

  If any check fails, **stop the whole plan** and report. Do not try to repair the
  checkout yourself; `sase workspace repair` is the owner tool.

### 2. Build outputs (regenerable)

Candidate Cargo target dirs. Enumerate them with `find`; do not hard-code workspace
numbers:

- `find ~/projects/github -mindepth 3 -maxdepth 3 -type d -name target`
- `find ~/Library/Application\ Support/sase/workspaces -mindepth 4 -maxdepth 4 -type d -name target`

Delete a candidate `T` only when all of these guards hold:

1. `T/CACHEDIR.TAG` exists and contains `Signature: 8a477f597d28d172789f06886806bc55`.
2. `$(dirname T)/Cargo.toml` exists.
3. Nothing in `T` was written in the last 60 minutes: `find "T" -mmin -60 -print -quit`
   prints nothing.
4. No running `cargo`, `rustc`, or `maturin` process has its cwd inside `$(dirname T)`.
   Check with `pgrep '^(cargo|rustc|maturin)$'` and `lsof -a -p <pid> -d cwd -Fn`.

When all four hold, run `rm -rf "T"`.

Expected targets:

- sase-core primary (13 GB);
- bob-cli primary at `~/projects/github/bbugyi200/bob-cli` (1.3 GB);
- actstat and zorg primaries;
- sase-core, bob-cli, and bbugyi200/bob-cli workspace checkouts (~7 GB total).

The primaries are rebuilt on their next `cargo build` or release, so the first build
afterwards is slower. That is the only cost. The installed `bob` is a copied binary at
`~/.cargo/bin/bob` that cron and the vault-sync LaunchAgent use; it is not affected.

Other build outputs:

- `~/tmp/sase/cargo-targets`, the legacy root. Delete it only when all of these hold:
  - `find ... -mmin -1440 -print -quit` is empty;
  - `env | grep -c 'tmp/sase/cargo'` is 0;
  - `grep -rIl 'tmp/sase/cargo' ~/.config/sase ~/.sase/projects/*/*.sase` finds nothing.

  Remove only the `cargo-targets` subdirectory, not `~/tmp/sase`.

- `~/projects/github/bbugyi200/bob-mac-capture/.build`. Delete it only when
  `pgrep -fl bob-mac-capture/.build` finds nothing and nothing in it was written in the
  last 60 minutes.

### 3. Package-manager and app caches

- Homebrew: `brew cleanup --prune=all`. Do not run `brew autoremove`, `brew uninstall`,
  or `brew upgrade`.
- uv: `uv cache clean`, **without** `--force`. If it reports an in-use cache or waits
  more than about 2 minutes, cancel and run `uv cache prune` instead.
- npm: run `npm cache clean --force` only if `pgrep -fl 'npm (install|ci)'` shows no
  install in progress. Otherwise skip. `--force` is npm's required confirmation flag,
  not an override of in-use checks.
- `~/Library/Caches/com.pinterest.PINDiskCache.PINCacheShared`: delete only when
  `lsof +D "<dir>"` is empty and `find "<dir>" -mtime -30 -print -quit` is empty.
- Do **not** clear Chrome's cache or model folders. Those are manual follow-ups.

### 4. Verified-redundant copies (irreversible deletions)

a. **Cutover rehearsal restore.** Delete
`~/cutover-backups/restores/kellys_mbp-20260906T172717-0aec9e-20260906T173029-42893e`
only when all of these hold:

- the matching backup `~/cutover-backups/backups/kellys_mbp-20260906T172717-0aec9e`
  exists;
- `cd` into it and run `shasum -a 256 -c SHA256SUMS --quiet`; it passes.

Keep `~/cutover-backups/sase-x7.6-cutover-*`, `~/cutover-backups/runs`, and
`~/cutover-backups/sase-x7-2-1-5-1`. They are small evidence files.

b. **Accidental research clone.** Delete `~/tmp/git@github.com:sase-org`, which contains
only the `sase--research` clone, only when all of these hold for that clone:

- `git status --porcelain --ignored` lists nothing except `.DS_Store` files;
- `git stash list` is empty;
- `git log --branches --not --remotes --oneline` is empty.

c. **Stale vault clone.** Delete `~/projects/github/bbugyi200/bob` only when all of
these hold:

- It is not listed in the vault paths of
  `~/Library/Application Support/obsidian/obsidian.json`.
- None of these reference the path `projects/github/bbugyi200/bob` (excluding the
  `bob-cli` and `bob-mac-capture` siblings): `crontab -l`, `~/Library/LaunchAgents`,
  `~/.config`, `~/bin`, `~/.local/share/chezmoi`, `~/.sase/projects/*/*.sase`.
- `git status --porcelain --ignored` lists nothing except `.DS_Store` files.
- `git stash list` is empty and `git log --branches --not --remotes` is empty.
- For every local branch tip, `git -C ~/bob cat-file -e <sha>` succeeds, so the tip
  exists in the live vault.

### 5. Compress backups that must be kept (lossless, reversible)

Archive each of these directories:

- `~/cutover-backups/backups/kellys_mbp-20260906T172717-0aec9e`
- `~/var/backups/bob-pre-gitsync-macbook-20260827T125528`
- `~/tmp/old_sase`

For each `D`:

1. Archive it next to itself:
   `tar -C "$(dirname D)" -cf - "$(basename D)" | zstd -T0 -10 -o "D.tar.zst"`.
2. Verify the archive:
   - `zstd -t "D.tar.zst"` passes;
   - the file count from `zstd -dc "D.tar.zst" | tar -tf - | grep -vc '/$'` equals
     `find "D" \( -type f -o -type l \) | wc -l`.
3. Compare sizes. If the archive is at least 20% smaller than `/usr/bin/du -sk "D"`,
   remove `D` and write a one-line `D.README` next to the archive. It should explain how
   to restore: `zstd -dc D.tar.zst | tar -xf - -C <parent>`, and give the original size.
4. Otherwise, keep `D`, delete the archive, and record "kept, compression not
   worthwhile".

Do these archives last. Earlier steps free the temporary space they need.

### 6. Verify and report

- `df -k /System/Volumes/Data`: report free space before and after, and the delta. APFS
  clones make `du` totals overstate real savings, so `df` is authoritative.
- `sase doctor 2>&1 | tail -3`: there must be no new ERROR or WARN versus baseline. If
  there are, investigate and report them.
- `sase disk list` still works, and `bob --version` still works.
- Final response:
  - the per-item table (item, size, action, reason) and total reclaimed;
  - every skip and its failed guard;
  - the "Manual follow-ups" list below, carried over verbatim so the user sees it.

## Manual follow-ups (report only; the implementer must NOT do these)

1. **Kelly's account is probably the biggest consumer.** About 220 GiB of the used space
   is invisible to `bbugyi`, and most of it is likely `/Users/kellydonnelly`. Kelly can
   log in and review it in System Settings → General → Storage: their Trash (63 items),
   Downloads (1,128 items), Desktop, and Photos/Movies.
2. **Empty your own Trash.** macOS privacy protection hides `~/.Trash` from agents, so
   its size is unknown.
3. **Chrome on-device AI model (4 GB)** at
   `~/Library/Application Support/Google/Chrome/OptGuideOnDeviceModel`. Turn off
   on-device AI in Chrome's settings (Settings → System) so Chrome deletes it and
   doesn't download it again. Deleting the folder alone just triggers a re-download.
   Separately, Chrome's cache is 1.5 GB; you can clear it with Delete browsing data →
   Cached images and files.
4. **Install the pending macOS 26.7 update** once space is freed. The staged update is
   bloating Preboot (18 GB) and `/Library/Updates` (1.1 GB), and a previous attempt
   failed. macOS 27 is also offered (about 12 GB download); that is a separate choice.
5. **Shared apps (33 GB in `/Applications`)**, only with Kelly's agreement. For example:
   - PortraitPro Trial and PortraitPro Studio are both installed (1.6 GB each);
   - Adobe Photoshop 2023 (4.7 GB) plus 2.9 GB of Adobe support files;
   - GarageBand and Logic content (1.8 GB) if unused.
6. **Personal data that is yours to judge:**
   - `~/org` (5.7 GB; `lit_review`, `lib`), `~/bob/old_lib` (1.6 GB), and `~/Downloads`
     (1 GB);
   - reMarkable desktop container (2.5 GB), Clipy history (1.3 GB), and Wispr Flow (1.1
     GB).
7. **Security note, unrelated to space:** `~/tmp/bryanbugyi34-private.key` is a private
   key sitting in a scratch dir. Consider moving it into `pass` or the keychain.

## Expected outcome

Free space should rise from about 23 GB to about 75–85 GB. Actual figures depend on APFS
clone sharing and compression ratios.

Nothing user-facing should change:

- SASE workspaces, `bob`, cron jobs, the vault, chezmoi, and every app keep working;
- the only side effect is slower first builds of the Rust repos.
