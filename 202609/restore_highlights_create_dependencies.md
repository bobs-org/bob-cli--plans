---
tier: tale
title: Restore Highlights PDF rendering dependencies
goal:
  The installed `bob highlights create` command renders a valid marked PDF on this Mac
  with the minimal required font and TeX packages.
size: small
proposed_by: bbugyi200.kellys_mbp.9
create_time: 2026-09-03 05:21:13
status: wip
---

# Restore `bob highlights create` dependencies on this Mac

## Goal

Make the installed `bob highlights create` command render Highlights-ready PDFs on this
machine by installing the minimal missing runtime dependencies, without changing the Bob
vault, replacing BasicTeX, or modifying `bob-cli` source.

## Confirmed diagnosis

The command's primary executables are already installed and discoverable:

- `bob 0.1.0` is `/Users/bbugyi/.cargo/bin/bob`.
- Pandoc 3.11 is `/opt/homebrew/bin/pandoc`.
- XeLaTeX is `/Library/TeX/texbin/xelatex`, supplied by the Homebrew `basictex`
  2026.0301 cask.

`bob highlights create` invokes Pandoc with XeLaTeX, loads `fvextra.sty`, and selects
`DejaVu Serif`, `DejaVu Sans`, and `DejaVu Sans Mono`. Both additional requirements are
absent:

- `kpsewhich fvextra.sty` returns nothing, and `tlmgr info --only-installed fvextra`
  reports `installed: No`.
- The `font-dejavu` cask is not installed. Exact Fontconfig queries do not list the
  three requested DejaVu families, and `fc-match` substitutes Times New Roman, Verdana,
  and Andale Mono.
- A PDF-mode Pandoc probe using Bob's requested DejaVu families fails in `fontspec` with
  `The font "DejaVu Serif" cannot be found`. Repeating the probe with fonts that are
  installed gets farther and then fails with `File 'fvextra.sty' not found`.

`bob highlights doctor` currently reports only that Pandoc exists, so its successful
result does not contradict the render failures and is not sufficient validation for this
repair.

## Implementation

1. Recheck the relevant package-manager state immediately before mutation. Confirm that
   `pandoc`, `xelatex`, and the Homebrew `basictex` cask are still present, while
   `font-dejavu` and `fvextra` are still absent. If the state changed, install only what
   remains missing.
2. Install the DejaVu fonts with Homebrew's dedicated cask:

   ```bash
   HOMEBREW_NO_AUTO_UPDATE=1 brew install --cask font-dejavu
   ```

   Do not replace BasicTeX with the much larger MacTeX distribution. Refresh the
   Fontconfig cache only if exact post-install family queries still cannot see the new
   fonts.

3. Install `fvextra` into the existing system TeX Live 2026 tree with its package
   manager:

   ```bash
   sudo /Library/TeX/texbin/tlmgr install fvextra
   ```

   Let `tlmgr` resolve any TeX-package dependencies. Do not run a broad TeX Live upgrade
   unless `tlmgr` specifically requires one to install this package; if it does, stop
   and report the incompatibility rather than changing the distribution beyond this
   plan.

4. Verify the installed assets directly:
   - `kpsewhich fvextra.sty` resolves to a real file in the active TeX tree.
   - Exact `fc-list`/`fc-match` checks resolve `DejaVu Serif`, `DejaVu Sans`, and
     `DejaVu Sans Mono` to those families rather than fallback fonts.
   - `pandoc --version` and `xelatex --version` still succeed from the same paths.

5. Run an end-to-end smoke test through the installed `bob`, using this repository's
   `README.md` as the Markdown source and a unique `mktemp -d` directory as an explicit
   external `--output` target. This input contains headings and fenced code, so the
   render exercises the XeLaTeX engine, DejaVu fonts, and `fvextra` path together.
   Require `bob highlights create` to exit zero, report a positive page count, and
   create a nonempty PDF. Then run `bob highlights marker` on that PDF and require the
   embedded marker to contain the expected title, default `ready` status, and default
   `obsidian_ref` parent. Remove only the uniquely created temporary directory after
   those checks.
6. Rerun `bob highlights doctor` as a final regression check. Confirm that the vault
   remains clean and that the smoke test wrote nothing under `~/bob/xlib` or
   `~/bob/lib`. No repository or vault files should be changed by this plan.

## Validation and acceptance

The repair is complete only when all of the following are true:

- Homebrew records `font-dejavu` as installed and TeX Live records `fvextra` as
  installed.
- XeLaTeX resolves all three DejaVu families and `kpsewhich` resolves `fvextra.sty`.
- The installed `bob highlights create` successfully generates the disposable PDF,
  including its outline/page content and Bob's readable page-1 marker.
- `bob highlights doctor` still succeeds, the Bob vault's Git worktree remains clean,
  and no test PDF or sidecar remains in the vault.

If the smoke render fails after both dependencies resolve, preserve the full Pandoc /
XeLaTeX diagnostic, remove the disposable test directory, and stop without adding
unplanned packages; that result would falsify the scoped dependency diagnosis and needs
a new investigation.
