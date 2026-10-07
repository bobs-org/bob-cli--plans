---
tier: tale
title: Fix bob ref create Markdown renders failing on missing LaTeX packages
goal: Markdown PDF renders succeed on apollo again, and bob reports any missing LaTeX
  package up front in bob ref doctor, and on render failure, with the exact install
  command.
size: medium
proposed_by: bbugyi200.apollo.5o
status: done
---

# Plan: Fix `bob ref create` Markdown renders failing on apollo (missing LaTeX packages)

## Diagnosis

**Symptom.** On apollo, the research-highlights file hook ran
`bob highlights create --include-id <research note>.md` at 2026-10-07 21:45 UTC and
failed
(`~/.sase/file_hooks/runs/ca930965d0e3949557768480-0000-research-highlights.log`):

```text
bob ref: error: pandoc failed while rendering …/plan_decisions_cross_surface_ux.md (exit 43):
Error producing PDF.
! LaTeX Error: File `needspace.sty' not found.
```

**Trigger.** Commit `d4fab34` ("render paired return links in Markdown PDFs", landed
with `39915c5`) added `src/native/highlights_ref/return_links.tex`, which
`render_temp_pdf` (`src/native/highlights_ref/create.rs`) injects into _every_ Markdown
render through `header-includes`. Its first two lines are `\usepackage{needspace}` and
`\usepackage{tikz}`. The installed `~/.cargo/bin/bob` was rebuilt at 20:45 UTC with that
change. The last successful hook run was at 20:04 UTC and the first failure at 21:45
UTC.

**Environment.** apollo's TeX is a minimal TinyTeX (`~/.TinyTeX`, TeX Live 2026,
installed 2026-09-17, not managed by chezmoi). `kpsewhich` resolves neither
`needspace.sty` nor `tikz.sty` (TeX Live package `pgf`), so installing needspace alone
would just fail next on tikz. It also lacks `soul.sty`, which pandoc's LaTeX template
loads for any document with strikethrough or underline. That is a latent failure of the
same kind for future research notes.

**Root cause.** bob's Markdown render depends on a set of LaTeX packages that is
declared nowhere and checked before nothing. `bob ref doctor` checks only that pandoc
exists. It checks neither xelatex nor packages. A failure shows up late, inside a
file-hook log, as raw LaTeX output with no fix. apollo has hit this class of failure
three times:

- Sep 12–17: `xelatex not found`, until TinyTeX was installed.
- Sep 18: `setspace.sty not found`, followed by an ad hoc
  `tlmgr install setspace fvextra lineno upquote` (per
  `~/.TinyTeX/texmf-var/web2c/tlmgr.log`).
- Oct 7: `needspace.sty` (and, hidden behind it, `tikz.sty`).

The change was authored on athena (full TeX Live), where it passes. On apollo,
`cargo test --lib xelatex` fails both real-render tests
(`listen_card_xelatex_render_uses_only_existing_packages`,
`return_link_xelatex_render_pairs_pills_with_destinations`) with the same
`needspace.sty` error, so `just check` is also red there.

**No backfill needed.** The PDF for this note already reached the vault from the Mac
(vault commit `086d0e54`, `lib/chat/plan_decisions_cross_surface_ux.pdf`).

## Decisions

- **Keep `needspace` and `tikz`.** Both are standard TeX Live packages, and the
  return-pill design needs tikz. Inlining needspace's macro would remove one package but
  not the class of failure. Rejected.
- **Fix the machine, then make the dependency explicit in bob:**
  - a declared package list,
  - a `bob ref doctor` check that names every missing package with one copy-pasteable
    install command,
  - a `hint:` line on render failures.
- **Doctor rows are warnings, never failures**, like the existing `pandoc` row. Markdown
  rendering is optional per host.
- **The real-xelatex unit tests keep failing (not skipping) when packages are missing.**
  They are the regression net for exactly this problem.
- **No new subcommands or options.**

## Step 1: Repair apollo's TinyTeX (machine fix, no repo change)

`tlmgr` currently refuses installs until it updates itself:

```bash
tlmgr update --self
tlmgr install needspace pgf soul
kpsewhich needspace.sty tikz.sty soul.sty   # all three must resolve
```

TinyTeX is user-owned, so no sudo. If Step 2's doctor row or the Verification render
probe reports anything else missing, install that too with `tlmgr install`.

## Step 2: Declare and check the render's LaTeX packages

Add `src/native/highlights_ref/render_tex.rs` (register it in
`src/native/highlights_ref/mod.rs`).

1. `RENDER_TEX_PACKAGES: &[TexPackage]`, where each entry has `file` (e.g.
   `"needspace.sty"`) and `tlmgr` (the TeX Live package name). Use grouped comments.
   Names below were verified with `tlmgr search --file`:
   - bob's own headers (`PANDOC_HEADER_INCLUDES` and `return_links.tex`):
     `fvextra.sty`→`fvextra`, plus fvextra's non-core dependencies `lineno.sty`→`lineno`
     and `upquote.sty`→`upquote`; `needspace.sty`→`needspace`, `tikz.sty`→`pgf`.
   - pandoc's LaTeX template under bob's flags (xelatex, `linestretch`, `geometry`,
     `colorlinks`, highlighting): `amsmath.sty`→`amsmath`, `amssymb.sty`→`amsfonts`,
     `setspace.sty`→`setspace`, `iftex.sty`→`iftex`, `unicode-math.sty`→`unicode-math`,
     `fontspec.sty`→`fontspec`, `lmodern.sty`→`lm`, `xcolor.sty`→`xcolor`,
     `geometry.sty`→`geometry`, `fancyvrb.sty`→`fancyvrb`, `framed.sty`→`framed`,
     `hyperref.sty`→`hyperref`, `bookmark.sty`→`bookmark`.
   - content-dependent template packages: `longtable.sty`→`tools`,
     `booktabs.sty`→`booktabs`, `etoolbox.sty`→`etoolbox`, `footnote.sty`→`mdwtools`
     (tables); `graphicx.sty`→`graphics` (images); `soul.sty`→`soul`
     (strikethrough/underline).
2. `fn missing_packages(kpsewhich_stdout: &str) -> Vec<&'static TexPackage>`.
   `kpsewhich a.sty b.sty …` prints one path per _found_ file. Match by file name and
   return the unmatched entries in list order.
3. `fn install_command(missing: &[&TexPackage]) -> String` returns
   `tlmgr install <names>`, deduplicated and in list order.
4. `pub(super) fn append_tex_doctor_rows(warnings: &mut Vec<String>)`:
   - Find `xelatex` with `crate::native::env::find_on_path("xelatex")`. pandoc's
     `--pdf-engine=xelatex` also resolves it from PATH.
     - Found: row `xelatex: available (<path>)`.
     - Not found: row `xelatex: warn (command not found)` and warning
       `xelatex command not found; Markdown PDF creation is unavailable`.
   - Look up `kpsewhich` next to xelatex's path first, so it queries the same TeX tree,
     then fall back to PATH.
     - If xelatex is missing: `latex_packages: skipped (no xelatex)`.
     - If kpsewhich is missing: `latex_packages: warn (kpsewhich not found)` plus a
       warning.
   - Run kpsewhich once with every file. Its non-zero exit when something is missing is
     expected. Only a spawn failure counts as an error (warn row).
   - Rows:
     - all found: `latex_packages: ok (<N> checked)`.
     - some missing: `latex_packages: warn (missing needspace.sty, tikz.sty)` with
       warning
       `LaTeX packages missing for Markdown PDFs: needspace.sty, tikz.sty; install with: tlmgr install needspace pgf`.
   - Never push to `failures`, so the doctor result stays `ok`.
5. Call it from `src/native/highlights_ref/doctor.rs` right after the `pandoc:` row.
6. Update the `bob ref doctor` help footer ("Checks vault paths, sidecars, PDF markers,
   Git state, and optional ob support.") to mention the Markdown render tools (pandoc,
   xelatex, LaTeX packages).

## Step 3: Actionable hint when a render fails

- In `render_tex.rs`, add
  `pub(super) fn render_failure_hint(detail: &str) -> Option<String>`.
- In `render_temp_pdf`'s failure branch (`create.rs`), append `"\nhint: {hint}"` to the
  existing error message when it returns `Some`. The `create` error printer already
  splits on `\nhint: ` and prints a separate `hint:` line. Keep the rest of the message
  byte-identical.
- Cases:
  - pandoc output contains ``! LaTeX Error: File `<file>' not found.`` and the file is
    in `RENDER_TEX_PACKAGES` →
    ``missing LaTeX package <file>; install it with `tlmgr install <pkg>`, then run `bob ref doctor` to check the rest``.
  - The same error for an unlisted file →
    ``missing LaTeX file <file>; find its package with `tlmgr search --global --file /<file>`, then run `bob ref doctor` ``.
  - pandoc's `xelatex not found` (exit 47) →
    ``install a TeX distribution that provides xelatex (e.g. TinyTeX), then run `bob ref doctor` ``.
  - Anything else → `None`.

## Step 4: Tests

Unit tests in `render_tex.rs`:

- **Drift guard.** Every package named by `\usepackage[…]{a,b}` in
  `create::PANDOC_HEADER_INCLUDES` and `return_links::HEADER_INCLUDES` must have
  `<name>.sty` in `RENDER_TEX_PACKAGES`. Widen `PANDOC_HEADER_INCLUDES` to `pub(super)`
  if needed. Adding a header package without declaring it then fails CI on every
  machine, not just at render time on a minimal one.
- `missing_packages` on sample kpsewhich stdout, where some files are found, returns the
  rest in order. `install_command` deduplicates the tlmgr names.
- `render_failure_hint`:
  - the verbatim stderr from the hook log above → the `tlmgr install needspace` hint;
  - an unlisted `.sty` → the `tlmgr search` hint;
  - the `xelatex not found` text → the TeX distribution hint;
  - an unrelated error → `None`.

CLI test in `tests/cli/highlights/marker.rs`, next to
`highlights_ref_doctor_checks_vault_git_without_writes`, reusing its vault setup:

- Put stub `xelatex` and `kpsewhich` scripts in a stub bin directory prepended to PATH.
  The kpsewhich stub echoes a path for every argument except `needspace.sty` and
  `tikz.sty`, then exits 1.
- Assert stdout contains `xelatex: available`,
  `latex_packages: warn (missing needspace.sty, tikz.sty)`,
  `tlmgr install needspace pgf`, and `result: ok`.
- Existing doctor tests must keep passing whatever TeX the host has. They do not assert
  the new rows.

## Step 5: Docs

- `README.md` requirements bullet ("`pandoc` and `xelatex` for `bob ref create` Markdown
  targets…"): say that the render also needs the LaTeX packages `bob ref doctor` checks,
  and that doctor prints the `tlmgr install` line for any missing ones.
- `README.md` `doctor` bullet: add xelatex and LaTeX packages to the checked list.
- `docs/highlights-ref-sync.md`:
  - doctor paragraph (~line 59): describe the new warning-level `xelatex` and
    `latex_packages` rows;
  - render-failure sentence ("a render failure includes pandoc's diagnostic output",
    ~line 148): mention the `hint:` line for missing LaTeX packages.

## Verification

1. `kpsewhich needspace.sty tikz.sty soul.sty` resolves all three.
2. `cargo test --lib xelatex` shows both real-render tests passing on apollo.
3. `cargo run --quiet -- ref doctor -n` prints `xelatex: available (…)` and
   `latex_packages: ok (…)`.
4. End-to-end render outside the vault with the workspace build:
   - In a fresh scratch dir, write `probe.md` containing a same-document link
     (`[jump](#later)` plus a `## Later` heading), a pipe table, `~~strikethrough~~`,
     and a fenced code block.
   - Run `cargo run --quiet -- ref create -n -o <scratch>/probe.pdf <scratch>/probe.md`.
     It must succeed and print a `links:` summary.
   - `git -C ~/bob status --short` must be unchanged.
5. `just check` passes.

## Out of scope

- DejaVu font availability: renders already succeed with them on apollo.
- Provisioning TinyTeX reproducibly through chezmoi: it is not managed there today.
  Mention it in the final summary as a possible follow-up rather than editing the
  chezmoi repo here.
