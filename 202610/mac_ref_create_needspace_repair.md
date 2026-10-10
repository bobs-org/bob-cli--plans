---
tier: tale
title: Repair Mac Markdown reference creation missing needspace
goal:
  Restore Markdown PDF rendering on the Mac, recover the failed research capture, and
  document the verified BasicTeX recovery procedure.
size: small
proposed_by: bbugyi200.apollo.6j
create_time: 2026-10-10 15:18:54
status: wip
---

# Repair the Mac's missing LaTeX dependency for `bob ref create`

## Outcome and scope

Restore Markdown PDF creation on Bryan's MacBook, recover the failed research capture,
and document the verified macOS recovery procedure. This is a small tale: one agent can
repair one host and update one documentation section. The existing renderer, dependency
manifest, diagnostics, and file-hook configuration already implement the intended
behavior.

Implementation starts only after this plan is approved. During planning, all
investigation was read-only apart from SASE audit/workspace bookkeeping and this scratch
plan. No packages, application files, configuration, or vault content were changed.

## Diagnosis and evidence

The supplied `~/tmp/bob_ref_create_error.txt` records the research file hook running
`bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT"` with project `bob-cli` and
source basename `apollo-pass-pinentry-hang.md`. Pandoc exited 43 after XeLaTeX reported
`! LaTeX Error: File 'needspace.sty' not found` (the actual TeX diagnostic uses an
opening backtick). Bob exited 1 and printed the appropriate package hint. The preceding
`--highlight-style` deprecation warning was not the fatal error.

Read-only SSH checks on `mac` as `bbugyi` confirmed:

| Check                       | Observed result                                                                                                                   |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Installed Bob               | `~/.cargo/bin/bob`                                                                                                                |
| Pandoc                      | `/opt/homebrew/bin/pandoc`, version 3.11                                                                                          |
| XeLaTeX, kpsewhich, tlmgr   | `/Library/TeX/texbin/`, BasicTeX / TeX Live 2026                                                                                  |
| System TeX tree             | `/usr/local/texlive/2026basic`, owned by `root:wheel`                                                                             |
| `bob ref doctor --no-hooks` | `latex_packages: warn (missing needspace.sty)`; `writes: none`; `result: ok`                                                      |
| Other declared packages     | All other 23 packages resolve, including `tikz.sty`, `fvextra.sty`, `lineno.sty`, and `upquote.sty`                               |
| Fonts via `fc-match`        | Actual DejaVu Serif, DejaVu Sans, and DejaVu Sans Mono matches                                                                    |
| Default `TEXMFHOME`         | `/Users/bbugyi/Library/texmf`, already in kpsewhich's search path; no user package database yet                                   |
| Remote package metadata     | `tlmgr info --only-remote --data name,relocatable,depends needspace` returned `needspace,1,`: relocatable, no listed dependencies |

The causal chain is established in the current source:

1. Commit `d4fab34` added return-link rendering. The header
   `src/native/highlights_ref/return_links.tex` unconditionally loads `needspace` and
   `tikz`; `render_temp_pdf` in `create.rs` includes it for every Markdown render. A
   report does not need return links to encounter this missing package.
2. Commit `73f9cc4`, implementing `plan:202610/ref_create_latex_packages.md`, added the
   package manifest, doctor checks, render-failure hints, and regression tests. Those
   changes are already present, including in the Mac's installed Bob. That plan targeted
   apollo's separate TinyTeX installation; installing a package there does not install
   it on the Mac.
3. The Mac's BasicTeX package set still lacks `needspace`. This is a host dependency
   gap. The error log and independent package lookup agree; there is no evidence of a
   quoting, project-routing, or missing-renderer problem.

The linked chezmoi repository's `home/dot_config/sase/sase.yml` contains the expected
hook command. Inspection found no managed TeX provisioning to amend. Keep this repair in
the existing TeX installation and Bob's prerequisites guide.

## 1. Recheck the host and install the missing package

Read the applicable `tailnet.md` and `obsidian.md` reference memories through
`sase memory read` before host/vault operations. Use SSH alias `mac` with bounded
connection timeouts; the Mac can be offline with its lid closed. Preserve a short
before/after record of tool paths, versions, package lookups, and doctor rows.

Reconfirm that the Bob/file-hook execution user and TeX tools match the evidence above.
Check the hook environment where available, especially `PATH`, `HOME`, `TEXMFHOME`, and
`BOB_PANDOC_COMMAND`, without dumping unrelated environment data. The initial live
checks used a login shell; verify visibility in the hook's execution environment as part
of the final retry.

Use the supported user-mode install as `bbugyi`, with the `tlmgr` belonging to the
active XeLaTeX installation. The package is relocatable and the default user tree is
already searched, so no administrator access or custom PATH/TEXINPUTS setting is needed
for the planned repair:

```sh
set -eu
tex_home=$(/Library/TeX/texbin/kpsewhich -var-value=TEXMFHOME)
if ! test -f "$tex_home/tlpkg/texlive.tlpdb"; then
    /Library/TeX/texbin/tlmgr init-usertree
fi
/Library/TeX/texbin/tlmgr --usermode install needspace
/Library/TeX/texbin/kpsewhich needspace.sty
```

Check each command's success and stop on failure. Preserve any pre-existing user tree
and its contents; skip installation if the file now resolves. Do not copy a raw `.sty`
into the repository or vault. Do not reinstall BasicTeX, install full MacTeX, or update
every TeX package for this one missing dependency.

If the package manager reports a repository/year mismatch, required manager update, or
unsupported user-mode operation, diagnose that exact error first. A system-tree fallback
is allowed only if user mode cannot solve the observed problem. Any privileged repair
must use `/sase_sudo` with `machine: mac`, exact reviewed argv, and an explanation of
why root is necessary; never raw sudo or a password request. The targeted system install
would use `/Library/TeX/texbin/tlmgr install needspace`. Do all non-root preparation
first. Use `/sase_monitor` for commands requiring a long-running handoff and wait for
the handoff command to exit as instructed by that skill.

## 2. Prove actual Markdown rendering works on the Mac

Run `bob ref doctor --no-hooks` again. Require the specific
`latex_packages: ok (24 checked)` row for the current manifest, together with available
pandoc and XeLaTeX. A successful doctor exit alone is insufficient: missing optional
render dependencies intentionally produce warnings. Unrelated existing library/listen
warnings are outside this repair.

Create a disposable probe under a fresh scratch directory on the Mac, with a space in
its path. Include a same-document link and matching heading, a pipe table, a fenced code
block, and strikethrough. Render with the installed Bob:

```sh
bob ref create --include-id -P bob-cli --no-audio \
    --output "$probe_dir/probe.pdf" "$probe_dir/probe.md"
```

Require exit 0, a nonempty readable PDF with pages, and a successful paired-link
summary. This exercises Bob's actual filters/header, fonts, XeLaTeX, PDF stamping,
project argument, and paths containing spaces. Keep the probe outside the vault and
remove only its own scratch files after verification. Do not use Markdown `--dry-run` as
a rendering test: it returns before invoking pandoc.

## 3. Recover the failed research capture

Recover the source using its durable reference
`research:202610/apollo-pass-pinentry-hang.md` in the `bob-cli` project context on the
Mac. Consult `sase_artifacts.md` before artifact operations, use `sase artifact read`
for audited content access and `sase artifact path` for the path supplied to Bob. Use
`/sase_repo` to open any other repository needed for recovery; never reuse a numbered
workspace path from this plan or browse sidecar files directly.

The planning host could not resolve this research reference: its audited read returned
`missing`. Its presence on the Mac or in durable storage remains to be verified. Resolve
there before assuming it is lost; if it is unavailable, report the exact recovery
blocker without recreating the report from its title or claiming the original capture
succeeded.

Check whether the reference is already queued or captured before writing. If absent,
preview the resolved source with the original `--include-id -P bob-cli` arguments, then
run the capture with those arguments in the corresponding hook environment. Quote the
resolved source path. Reuse a supported SASE retry of the specific failed hook if
available, or replay its command with the same project and source; record which was
tested. Do not run both and create duplicate work.

Require the intended intake PDF to exist and Bob to report the expected source id and
resolved parent. If already captured, verify the existing reference and use the scratch
render as proof of the dependency repair; do not force an overwrite. Capturing produces
an intake PDF for later scanning. A broad `bob ref scan`, rewriting the research note,
or touching other failed hooks is outside this repair.

## 4. Document the verified recovery procedure

Update the prerequisites section in `docs/highlights-create.md` with a concise macOS
BasicTeX troubleshooting subsection. Cover:

- Use `kpsewhich`, `xelatex`, and `tlmgr` from the same installation; on this setup they
  are under `/Library/TeX/texbin`.
- For a relocatable missing package such as `needspace`, initialize the default user
  tree once and install with `tlmgr --usermode`. These commands run as the same user as
  the hook. User-tree packages are maintained separately from the system installation;
  user mode must be specified on future package operations.
- Verify the `latex_packages` row and perform a real render. A doctor exit of zero or
  Markdown `--dry-run` alone does not establish render readiness.
- The `--highlight-style` warning is distinct from a fatal LaTeX missing-file error.
  Record the successful method actually exercised during this repair.

Link to the official
[TeX Live user-mode documentation](https://tug.org/texlive/doc/tlmgr.html#USER-MODE).
The [CTAN needspace entry](https://ctan.org/pkg/needspace) confirms its TeX Live package
name. Keep this as an operator procedure; no automatic package install during file-hook
execution, no new CLI options, and no changes to doctor's warning contract are needed.
The existing manifest and tests remain valid.

## Acceptance and final report

- `needspace.sty` resolves for the Mac's hook user through the active TeX tree.
- Doctor reports all declared render packages available.
- The real Mac probe produces a valid PDF and paired return links.
- The original capture succeeds or is confirmed already present; if its source cannot be
  recovered, clearly distinguish that remaining blocker from the verified renderer
  repair.
- The documentation describes commands verified on this host, and `git diff --check`
  passes. A Rust rebuild or new unit tests are unnecessary for the intended host/package
  and documentation-only change; the real Mac render is the relevant integration check.
- Report the installation location, before/after diagnostics, render result,
  original-hook recovery result, and any remaining concrete blocker. The deprecated
  highlighting flag may still warn; do not label it the root cause.
