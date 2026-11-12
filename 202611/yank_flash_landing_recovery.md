---
tier: tale
title: Recover yank-flash repairs and land bob-cli-kb
goal:
  Correct viewport rendering, establish reliable Obsidian acceptance, integrate newer
  plugins and docs, deploy the verified build, and close bob-cli-kb.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kb.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kb
create_time: 2026-11-12 10:47:01
status: wip
---

- **BEAD:**
  [bob-cli-kb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kb/README.md)

# Recover the remaining yank-flash repairs and land bob-cli-kb

Finish the remaining work of epic `bob-cli-kb` in one coding agent, including its normal
closeout. Both original phases are closed, but their current source does not satisfy the
accepted viewport and verification contracts. This tale continues
`plan:202611/yank_flash_landing_repairs.md` and adds the subsequently reported mounted
documentation link and newer plugin integration. Do not restart the feature or create
another epic. There is no separate land agent after this tale.

## Evidence and boundaries

Read `bob-cli-kb`, `bob-cli-kb.1`, and `bob-cli-kb.2` with `sase bead read`, including
all notes. Consume `plan:202611/vim_yank_flash.md` and
`plan:202611/yank_flash_landing_repairs.md` through `sase artifact read`. The epic has
no separate DECISIONS and no parent bead, confirmed by this landing audit. Preserve
original C1–C11 and the approved repair tale's explicit reporting of unsupported
clipboard verification. No memory changes are needed.

Open `bob-plugins` through `/sase_repo` and read its `AGENTS.md`. All implementation
paths below are relative to that checkout. Open the plans sidecar through `/sase_repo`
before changing its plan frontmatter; read artifact contents through audited commands.
No bob-cli product-code change is currently indicated.

Audit baseline: bob-cli `753b4e48`, bob-plugins `3dc30c7`, both equal their fetched
origin/master. The original plugin/test commits are `95e2e51` and `7197394`. `99ccca1`
subsequently clarified the vault's normal-mode Y mapping; preserve it. No later commit
on fetched plugin refs contains the previous landing repairs.

Fresh checks on that baseline:

- The yank-flash unit and offline suites pass 35/35, build:check passes, and validate
  passes inventory and 10/10 manifests. They do not catch the defects below.
- Executing the real `buildFlashDecorations` with `aaaa\nbbbb\ncccc`, charwise selection
  `[0,4)`, and visible range `[10,14)` returns the offscreen mark `[0,4)`. An empty
  visibleRanges array paints all three linewise rows.
- Executing the real e2e `assertFlash` accepts a fabricated yw mark on line 999
  containing WRONG TEXT when the register is alpha. Its yiw assertion also accepts EXTRA
  alpha EXTRA on the wrong line.
- `bob plugins list --repo <opened-checkout> --no-pull -f json` reports yank-flash 0.1.0
  synced, enabled, and clean at `3dc30c7`, with no vault-ignore warning.
- `sase bead epic-symbols bob-cli-kb` reports none; the host has no symvision recipe.

The historical report `file:explicit:9d4d95d992913e20d1fbc473` describes a repaired
0.1.1 build and stronger checks, but neither the current source nor the deployment
contains that build. Treat the report as historical evidence, not proof for this tree.
The saved chat resolved by
`sase chat show --basename gh_bobs_org__bob_cli-ace_run-bob_cli_kb_land__c-261111_161744`
records an approved overwrite of the then-protected 0.1.1 deployment and a staged sync,
not a completed source landing. Current clean provenance should permit ordinary sync. Do
not repeat that obsolete approval question. If recovering any prior implementation, use
audited artifacts or an allowed repository ref, never a sibling workspace path; verify
recovered code against current source and rerun its checks.

## 1. Bound decorations and expiry correctly

In `plugins/bob-vim-yank-flash/main.js`, intersect charwise and blockwise ranges with
actual `view.visibleRanges`. Empty or unavailable ranges must not broaden paint.
Preserve UTF-16 positions, omit empty intersections, and retain captured selection
ranges so scrolling can rebuild the visible portion during the same flash. A fully
visible charwise range remains one mark; blockwise rows retain separate marks.

Keep linewise whole-line coverage, blank lines, blank EOF, and de-duplication, but
traverse only visible intersections rather than enumerating every line of a huge yank
before filtering. No size cutoff, scrolling, unfolding, text search, key sniffing,
operator replacement, settings, or live user-text logging.

Add focused regressions for offscreen, partial, disjoint and empty viewports in all
three modes, viewport rebuilds, blank EOF, and large-yank traversal using a counted
operation assertion instead of a timing budget. Preserve exact selection matching,
original-first invocation with unchanged this/arguments/return, black-hole/non-yank
filters, per-controller wrapping, and safe unload behavior. Include positive hook
coverage for unnamed, named, append, plus and star registers.

The timer currently skips disconnected views. Reproduce a detach/wait/reattach case and
ensure an expired flash cannot reappear. Keep generation-safe 200 ms replacement and
safe destruction/unload; do not dispatch into a destroyed editor. Treat the live detach
issue as unverified until the new regression establishes it.

## 2. Make the e2e assertions establish the contract

Repair `scripts/obsidian-vim-yank-flash-e2e.cjs` and its offline test:

- Map rendered DOM back to numeric source offsets with the real CM view. The current
  `.cm-line[line.number-1]` lookup breaks in a scrolled viewport. Compute expected
  source spans independently from known fixture positions and compare exact normalized
  coverage, clipped to the observed viewport. Exclude widget DOM from source text.
- Compare copied bytes and the relevant registers exactly, including linewise trailing
  newline, named-register append, and blockwise separators. Add offline negative tests
  for wrong location/text, missing rows and extra coverage. A non-null register or
  substring/minimum-mark test is insufficient.
- Expand the 18-line fixture to genuinely scroll deep into a long note. Verify ygg clips
  to visible lines and preserves normal Vim scroll/fold behavior by comparison with the
  disabled plugin. Include inline-code visibility alongside search matches, tinted task
  rows, emoji, tabs, wrapping, and Live Preview cursor-line cases.
- Require exact expected paste results for yyp and yyP, compare enabled/disabled
  cursor/register/document outcomes, test undo after an actual yank/paste, and compare
  both 3yy notices. Retain dot-repeat and macro parity.
- Measure two yanks occurring inside the first 200 ms lifetime; prove the first timer
  cannot clear the second flash. Switch away and back after expiry, and retain split,
  unload/reload, and idle-clear coverage. Keep original gesture-matrix coverage.
- Preserve truthful pop-out/fold skips when the harness cannot establish those cases. As
  the approved prior repair tale permits, either exercise actual vimrc-support clipboard
  delivery or explicitly report it unverified. An unchanged Electron clipboard is not
  proof of copying. Never open the live vault in a test instance.

## 3. Integrate later changes and fix the docs

The lander reviewed all post-start commit subjects in both repositories and the relevant
source. Preserve the following integration points:

- Copy card (`353eedb`, `d1d294b`, `934bbbb`, `d532aad`, `c9d4572`) writes through
  `navigator.clipboard.writeText` in `641-plugin-copy-card.js`. It must copy without any
  Vim flash or document/cursor mutation. Retain path-yank and surround exclusions.
- Keep-moving marks (`27b99dd`, `4e1f377`, `e8a776e`) add CM widgets at side 3. Exercise
  a real rendered mark while yanking its source row; widget text must not enter the
  oracle or alter the register. Do not change its current project-eligibility rules.
- Bank marks (`4e4a580`, `a6f4b03`, `2f02824`) replace amount/category/account spans and
  add heading widgets. Album catalog/header work (`bd6fc41`, `ed5e648`, `c2f6834`) adds
  cards and inline widgets. Include representative current widget/replaced-span
  coexistence cases with source/register/cursor parity. C9 keeps replaced/widget paint
  best-effort; don't redesign these features or make all their visual chrome flash.
- `d94c6bb` split npm tests into unit and serial performance stages. The baseline has
  340 distinct explicit tests, including both yank-flash files in test:unit. Preserve
  every existing entry and the serial perf gate when adding oracle tests.
- Host `6a88d810` changed sync/list repository resolution and fetch fallback. Use
  explicit `--repo` and `--no-pull` for this deployment. Other reviewed host changes
  cover album/bank/capture/security/mail and do not introduce a direct flash consumer.

Refresh histories before final verification and account for further relevant drift. Read
docs reference memory before documentation edits. In `docs/bob-vim-yank-flash.md`, move
yap/yG/ygg into linewise examples, state visual-line whole-line behavior, and remove the
misleading claim that dot repeats a yank (the actual parity case is yyp.). Keep the
landed stock-Y versus vault-y$ clarification.

Fix `docs/develop/validation.md`'s `../bob-vim-yank-flash.md#verification` link at its
source. The host mounts this file at `develop/plugins/validation.md` but the target at
`reference/plugins/bob-vim-yank-flash.md`, so that relative path is invalid. Use the
existing development-doc convention of the absolute published URL
`https://bobs-org.github.io/reference/plugins/bob-vim-yank-flash/#verification`. This
resolves epic notes 2–5; it is epic work, not an unrelated docs baseline failure.
Preserve published paths and generated regions. Bump the plugin patch version for
production fixes and run `npm run docs:gen`.

## 4. Verify the repaired tree and deploy

Run focused regressions, `npm run build:check`, `npm test`, `npm run validate`, and
`BOB_CLI_DIR=<host-checkout> npm run docs:check`. Use `just check` in the host for
file-change verification. Never run `just check-full`. Diagnose new failures; separate
independently evidenced existing infrastructure failures from epic regressions without
weakening assertions. The mounted yank-flash link warning must be gone.

Run the full strengthened real-Obsidian check on athena:

```sh
xvfb-run -a node scripts/obsidian-vim-yank-flash-e2e.cjs \
  -e /snap/bin/obsidian \
  -r /tmp/bob-yank-flash-recovery.json \
  -o /tmp/bob-yank-flash-recovery-shots
```

Choose fresh output paths for repeats, inspect all eight light/dark screenshots, and
preserve the passing report as a SASE artifact tied to the epic. Record tested source
content hashes as well as HEAD when tests run before the final commit, so a historical
report cannot again be mistaken for proof of different bytes. Report actual per-group
passes and skips. Use `/sase_monitor` for long checks and carry all remaining steps in
its continuation; wait for the staging command itself to exit, then finalize.

After verification deploy only the repaired plugin with
`bob plugins sync --repo <opened-bob-plugins> --no-pull -p bob-vim-yank-flash`. Follow
the linked repo's refusal procedure if necessary; re-use existing authorization only
where it actually applies. Confirm enabled/synced status, exact managed bytes, and
vault-ignore protection. Do not commit directly in the vault. This deployment and
closeout must not depend on the repair's future commit SHA, push, or CI result.

## Follow-up dispositions already completed by the lander

Carry these into the close note. Existing reporter `bob-cli-kb.land` has already
corroborated all six tracked issues; do not repeat +1 for the same historical evidence.
The refreshed same-type searches, recent task sweeps, active-epic scope review and
audited candidate reads found no new causal owner for these pre-existing reports.

- kb.1 note 1 frees-marks timing: existing `bob-cli-io`, ready; off-clock chips:
  existing `bob-cli-ju`, ready. No new task or fresh full-suite failure claimed.
- kb.1 note 2 plus kb.2 note 1 four mounted docs budgets: `bob-cli-jz`, now closed done;
  protocol budget: `bob-cli-h9`, now closed done. Their notes identify fixes in Mac
  `2007294` and plugin `cb32d8d`. Do not reopen on pre-close evidence.
- kb.2 note 1 Block ID performance: `bob-cli-f8`, now closed done following production
  fix `70408bf` and serial-stage `d94c6bb`. Reduced-motion browser timeout remains
  `bob-cli-fz`, ready, with the prior qualified phase corroboration.
- kb.2 note 1 transient Relation Chips click assertion: decline a new task because the
  report lacks an exact failing node/trace and remains unreproduced. A fresh
  `test-ledger-tools-relation-chips.cjs` run passed 27/27, including MN click. Preserve
  the observation; do not conflate it with Chromium timeout causality.

Any newly discovered unrelated issue uses `/sase_new_task`; epic-caused issues stay in
this tale's implementation work. No new follow-up task was created during this audit.

## 5. Close bob-cli-kb and finish its plans in this tale

This is the final implementation step, not a later land-agent dependency. Re-read the
epic, both children and every note; recheck descendant readiness and both linked plans.
Confirm the repairs, current integration and verification satisfy the remaining scope.
Run `sase bead epic-symbols bob-cli-kb`. Resolve every listed exemption by wiring up,
privatizing, adding a legitimate non-test pragma or deleting its symbol; re-key only to
a still-open later bead that truly needs it. Then run:

```sh
sase bead close bob-cli-kb --note "<specific source/behavior verification, integration, test results and skips, deployment evidence, and every follow-up disposition above>"
```

If symbols reject the close, finish their cleanup and retry. If named phases are
incomplete, finish or reopen them. Never use --force merely to succeed or advance a
successful nested landing; canceled/superseded work requires an explicit deliberate
resolution and reason. After closing, run `just symvision` if available and record its
absence otherwise. In the opened plans repository, set `status: done` in
`202611/vim_yank_flash.md` (the epic's PLAN). Also mark
`202611/yank_flash_landing_repairs.md` done once its continued scope is verified;
preserve other frontmatter and body content.

Reconfirm with `sase bead read bob-cli-kb -r "Need the parent link"`. The concrete
audited result is no parent, so finish normally. If a new parent was attached during
implementation, inspect its ID and type: verify and close only a phase parent; for a
plan parent re-audit its previous landing note, all descendants/notes, linked plan and
post-child drift before symbol cleanup, normal close, symvision and plan status update.
Repeat only through fully complete directly parented plan ancestors. At the first
incomplete/ambiguous parent, note the exact blocker there and report it.

Finish through `/sase_final` with commit decisions for every changed repository,
including bob-plugins and plans. The final response must state the actual close result,
report artifact and skips. Remind Bryan that running Obsidian instances need restarting,
and the Mac needs `bob plugins sync` before restart; the requested manual light/dark
check is yy on a tinted task, vey, blockwise yank, and black-hole yank.
