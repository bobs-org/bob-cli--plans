---
tier: tale
title: Finish Vim yank flash viewport and verification repairs
goal: Prove exact yank feedback on current plugins and close bob-cli-kb normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kb.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kb
create_time: 2026-11-11 12:29:30
status: wip
---

- **PARENT:**
  [202611/vim_yank_flash.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/vim_yank_flash.md)
- **BEAD:**
  [bob-cli-kb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kb/README.md)

# Finish Bob Vim Yank Flash and land bob-cli-kb

Repair the remaining viewport and verification gaps in epic `bob-cli-kb`, then finish
its landing in this implementation turn. This is one bounded plugin/test/docs repair,
not another multi-phase epic. Do not wait for another land agent or for the final code's
future commit, push, or CI result to close the epic.

## Context and scope

Read `sase bead read bob-cli-kb -r "Need current landing notes and child readiness"` and
its two children, including all notes. Read the approved original plan with
`sase artifact read plan:202611/vim_yank_flash.md "Need the accepted yank-flash contract"`.
The original epic has no separate DECISIONS block and no `parent_bead` as of this audit.
Keep C1-C11 and the accepted pushText design. Never override the yank operator, sniff
keys, search the document for copied text, change editing behavior, add settings, or
retain/log live user text. No memory edits are needed or authorized by this tale.

Open `bob-plugins` through `/sase_repo` and read its `AGENTS.md`. All implementation
files named below are relative to that linked checkout. The host bob-cli checkout needs
no product-code change. The original plan lives in the plans sidecar at
`202611/vim_yank_flash.md`; open that repository through `/sase_repo` before editing its
frontmatter, and consume its content only through the audited artifact command. Never
embed a numbered workspace path in this plan or its successor artifacts.

Already verified by the lander:

- Phase commits `95e2e51` and `7197394` exist in bob-plugins, with the plugin, 34 unit
  tests, offline e2e wrapper, docs inventory, and deployment guidance. Both children are
  closed normally. Every child note was reviewed.
- Focused
  `node --test scripts/test-vim-yank-flash.cjs scripts/test-obsidian-vim-yank-flash-e2e.cjs`
  passes 35/35 on `7197394`.
- `npm run build:check` and `npm run validate` pass on that tree, including current docs
  inventory and all ten plugin manifests.
- The real-Obsidian phase reports a passing 1.15.0 run, with explicit pop-out and
  heading-fold skips. Its current assertions do not establish all of the claimed
  range/behavior contract; the evidence must be strengthened below.
- `bob plugins list -f json` reports `bob-vim-yank-flash` 0.1.0 synced and enabled. Its
  deploy provenance still identifies the phase's uncommitted source, so follow the
  normal sync refusal procedure if a repair deployment encounters it.
- No `--epic-symbol` entries existed for `bob-cli-kb`; the host currently has no
  `symvision` recipe. Recheck both at closeout.
- Fetched bob-cli and bob-plugins master. Audited bob-cli through `0b4483be` and
  bob-plugins through `7197394`; no unseen base-branch commits existed at that point.

## 1. Make decorations obey the viewport contract

In `plugins/bob-vim-yank-flash/main.js`, `buildFlashDecorations` filters linewise
decorations but ignores `visibleRanges` entirely for charwise/blockwise marks. The
lander ran the real helper against `aaaa\nbbbb\ncccc`, selection `[0,4)`, viewport
`[10,14)`: it returns the offscreen mark `[0,4)`. Passing an empty `visibleRanges` also
paints all three linewise rows because `isLineVisible` treats an empty array as
everything visible. Both conflict with original C8.

Intersect charwise/blockwise ranges with actual visible ranges, preserving precise
UTF-16 offsets and skipping empty intersections. A fully visible charwise selection
still produces one mark; a block produces one per nonempty visible row. Preserve
whole-line coverage for linewise yanks, including an empty last line. An empty viewport
produces no decorations; uncertainty must not broaden the paint. Keep the captured
selection ranges unchanged so viewport changes can rebuild the visible portion of the
same flash. Avoid enumerating a whole huge yank merely to throw away offscreen lines:
bound line-decoration traversal to visible intersections. No size cutoff and no
scrolling or unfolding.

Add meaningful unit cases for fully offscreen and partially visible charwise/blockwise
ranges, disjoint/empty visible ranges, viewport changes, blank EOF, and deduplication.
Use counted traversal assertions rather than a flaky wall-clock budget for a large
document. Preserve all existing hook, text-check, stale-clear, and unload guarantees.

## 2. Make the real-Obsidian oracle prove the promised behavior

Repair `scripts/obsidian-vim-yank-flash-e2e.cjs` and its offline test. The lander
evaluated its actual `assertFlash` helper without launching Obsidian:

- A record `{kind:'mark', line:999, text:'WRONG TEXT'}` with register `'alpha '` passes
  the `yw` expectation `{minMarks:1}`.
- A wrong-location mark containing `EXTRA alpha EXTRA` passes `yiw`'s substring check.
  The register is only checked for non-null, never compared to the flash.
- The fixture is only 18 lines, so its `ygg` is not the planned deep-scroll case.
  Observer baseline lookup uses `querySelectorAll('.cm-line')[line.number-1]`, which
  does not identify lines correctly once the first rendered line is not line 1.
- `behavior/yyp` and `yyP` only test that the original line still occurs, so a paste
  that did nothing passes. The 3yy parity case compares text/cursor/register/clipboard,
  but only tests the enabled notice and does not compare the disabled notice.
- Repeated-yank timing sleeps 150ms after a key helper that itself sleeps after each
  key, so the second yank may happen after the first 200ms flash expired. It does not
  establish the required overlap/stale-timer behavior.

Observe numeric source offsets and line numbers through the actual CM view/DOM mapping,
not rendered-node ordinal. Compare complete normalized observed source spans against
independently expected spans for each gesture, after clipping to the captured viewport.
Cross-check the copied bytes and relevant register contents, accounting explicitly for
linewise trailing newlines, named-register append, and blockwise row separators.
Duplicate DOM samples must not be mistaken for extra source coverage; wrong text, wrong
offsets, missing rows, and extra coverage must fail. Add offline negative oracle tests
demonstrating those failures; do not rely on token-presence self-tests.

Use a genuinely long fixture and prove `ygg` from a scrolled position: baseline mapping
works, only visible lines decorate, and the flash does not introduce scroll or unfold
changes beyond Vim's normal behavior. Add the original design's inline-code visibility
case alongside the search-match and tinted-row cases. Preserve the explicit, honest
pop-out/fold skip policy; a claimed fold pass must establish an actual fold.

Compare enabled/disabled outcomes for actual paste results, undo after a yank/paste, and
the 3yy notice, retaining dot-repeat and macro parity. Establish the second yank
occurred inside the first flash's measured 200ms lifetime and that the first timer
cannot clear the new flash. Switch away and back after the timer and verify no stale
flash returns; the current callback skips disconnected views, so investigate that
specific lifetime risk and fix it if reproduced without dispatching into destroyed
views. Treat a tab-detach risk as a hypothesis until the regression demonstrates it.

Keep the scratch vault isolated. The original harness installs only repository plugins,
so its unchanged Electron clipboard is not by itself proof of vimrc-support clipboard
delivery. Make the clipboard check exercise actual copying under the supported clipboard
setup, or explicitly report that part unverified rather than claiming parity from an
unchanged clipboard. If an external plugin source is needed, use `/sase_repo` first. Do
not access or open the live vault as a test instance.

## 3. Integrate later work, update docs, verify and deploy

The post-start bob-plugins commits were `27b99dd` (Keep-moving marks), `353eedb` and
`d1d294b` (Copy card). The Copy card calls `navigator.clipboard.writeText` directly; it
must remain outside Vim-yank flashes. Keep-moving marks add CM widgets at side 3; they
must not change the source range comparison or be mistaken for copied text. Add focused
real-Obsidian integration cases for Copy card copying (no flash, document unchanged) and
a yank on a row with a rendered Keep-moving mark. Install the current repo plugins as
the harness already does; don't redesign those features. Retain the legacy
`yank-relative-path` no-flash case.

The post-start host commits cover SVG rendering, album record/capture/completion,
Keep-moving docs, and Copy card config docs; no host consumer calls this plugin. Refresh
both histories before completion and incorporate any newly relevant changes. The merged
package test list already contains both flash tests and later suites; preserve every
entry.

Correct `docs/bob-vim-yank-flash.md`'s table: `yap`, `yG`, and `ygg` currently appear as
charwise examples although the e2e correctly expects linewise output. Describe
visual-line yanks as whole lines and avoid saying dot repeats a yank when the parity
scenario is `yyp.`. Preserve docs URLs and generated sections. Bump the plugin patch
version for production changes and regenerate the inventory with `npm run docs:gen`.
Read docs reference memory before editing docs.

Run focused tests, `npm run build:check`, `npm test`, `npm run validate`, and
`BOB_CLI_DIR=<host-checkout> npm run docs:check` as required by the original plan. Run
`just check` in the host for file-change verification. Never run `just check-full`. Run
the strengthened complete Obsidian e2e on athena under xvfb, with unique report and
screenshot paths, inspect the light/dark screenshots, and preserve report evidence as
SASE artifacts. Use `/sase_monitor` for long commands, carrying this tale's remaining
closeout work in the continuation; never wait for a staged successor inside the turn.
Known unrelated test/docs failures listed below are separate infrastructure work;
classify any new failures with evidence instead of weakening checks or silently skipping
epic regressions.

Deploy the repaired source using
`bob plugins sync --repo <opened-bob-plugins> --no-pull -p bob-vim-yank-flash`. Follow
linked `AGENTS.md` if sync refuses; never force an overwrite. Verify synced/enabled
status and ignore protection. No manual vault commit. No handoff dependency on the
repair's future SHA, push or CI result.

## Follow-up triage already owned by the lander

The lander records the final outcomes on `bob-cli-kb` before proposing this tale. Carry
those outcomes into the close note without creating duplicate tasks:

- `bob-cli-kb.1` note 1: frees-marks budget -> existing `bob-cli-io`; off-clock
  link-chip budget -> existing `bob-cli-ju`.
- `bob-cli-kb.1` note 2 and `bob-cli-kb.2` note 1: four mounted docs budgets -> existing
  `bob-cli-jz`; protocol budget -> existing `bob-cli-h9`.
- `bob-cli-kb.2` note 1: three Block ID Prompt budget failures -> existing `bob-cli-f8`;
  reduced-motion browser timeout -> existing `bob-cli-fz`.
- The transient Relation Chips click assertion is declined as a new task for lack of an
  exact failing node/trace or reproducible failure in the preserved phase note and
  transcript. The lander's current focused relation-chip suite passes 27/27. Do not
  invent a node ID or merge that assertion into the Chromium timeout task. The
  observation remains recorded on the epic for a future reproducible report.

## 4. Finish this epic's closeout yourself

This tale has no separate land agent. Once the repairs and verification above are done,
re-read `bob-cli-kb` and both phase beads with audited reasons, check all descendant
readiness and the original linked plan, and record which concrete gaps were repaired,
the post-start integration checks, verification results/skips, and every follow-up
outcome above.

Run `sase bead epic-symbols bob-cli-kb`. Resolve each exemption by wiring up,
privatizing, applying a legitimate non-test pragma, or deleting the symbol; re-key only
to a still-open later bead that actually needs it. Then close normally:

```sh
sase bead close bob-cli-kb --note "<specific verification, integration, and follow-up outcomes>"
```

If leftover symbols reject the close, clean them up and retry. If an unfinished named
phase rejects it, finish or reopen that phase. Never force a successful landing merely
to get past validation. After close, run `just symvision` if the recipe is available;
record its absence otherwise. Set `status: done` in the frontmatter of the original epic
plan `plan:202611/vim_yank_flash.md` in the opened plans sidecar, preserving all other
fields and content.

There was no parent at the audited start. Reconfirm with
`sase bead read bob-cli-kb -r "Need the parent link"`; if still absent, finish normally.
If a parent has appeared, follow the landing prompt's parent rules: verify and normally
close only a phase parent, or fully re-audit a plan parent's descendants, notes,
linked-plan readiness and post-child drift before its own symbol cleanup/normal close,
symvision and done status. Stop and note any incomplete or ambiguous parent.

Final user response: give the actual landing result and report artifact, the per-group
e2e results and explicit skips. Remind Bryan that running Obsidian instances need a
restart and the Mac needs `bob plugins sync` before restarting; the requested two-minute
Mac theme check is `yy` on a tinted task, `vey`, blockwise yank, and black-hole yank in
light/dark. Finish through `/sase_final` with commit decisions for every changed
repository, including bob-plugins and the plans sidecar.
