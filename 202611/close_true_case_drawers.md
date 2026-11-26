---
tier: tale
size: medium
title: Finish and close the true-case drawers epic bob-cli-ph
goal:
  The drawer tray's stroke (and, under widen, its width) is e2e-asserted against the
  requirement diamond with screenshots registered on bob-cli-ph, bob-ledger-tools is
  deployed, and epic bob-cli-ph is closed with a verification note.
decisions:
  tray_width:
    ask:
      Should the Obsidian drawer tray widen to about 0.9em across, as epic plan §7
      specifies?
    choices:
      widen:
        Redraw in a wide viewBox; stroke stays matched; all three hosts redeploy after
        the commit lands
      keep:
        Keep the 0.59em square-viewBox tray shipped in 0d0a31f; only apollo needs a
        deploy
    default: widen
    why:
      Approved §7 design is a wide, shallow tray ~0.95em across; today's 0.59em box
      nears the checkbox
    answer: widen
proposed_by: bbugyi200.apollo.g5
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 13:33:27
status: wip
---

# Finish and close epic bob-cli-ph (true-case drawers)

## Context

Epic `bob-cli-ph` (plan `plan:202611/true_case_drawers.md`) has all 11 phases closed.
Its lander tale `plan:202611/land_true_case_drawers.md` ran across several turns. Most
of that tale has landed. The epic is still `IN_PROGRESS` because three things never
happened:

- the tray e2e evidence,
- the apollo deploy,
- the close itself.

Do not redo the triage. Every phase follow-up was settled in the epic's LAND TRIAGE note
(#12).

Governing decisions:

- `decisions:drawers-keep-their-names`: a drawer is a whole-bold, non-checkbox child.
  ALL-CAPS is ordinary content.
- `decisions:mac-capture-is-a-thin-client`: Mac fixtures come from bob, never by hand.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit the `Cargo.toml` version
  or `CHANGELOG.md`. This work is CSS, test, and docs only, so no commit takes `!`.

### Already done (verified at plan time; re-confirm cheaply, don't redo)

- **Rust goldens.** Two bob-cli commits, `4dd787bc` and `8025ab3c`, migrated all 80
  drawer-cutover tests to bold true-case. Epic note #13 recorded the result: lib 8236
  passed / 0 failed, CLI 4260 passed / 0 failed. `bob-cli-q4` is closed, so no failures
  are allowed. No `src/` change was needed.
- **Mac fixtures.** bob-mac-capture `6ea5b8f6` regenerated the drawer fixtures.
  - Provenance: bob-cli `db5a70cc`, `bob 0.6.0`, grammar 33, `schema_version` 1.
  - No generated `Tests/Fixtures/*.json` contains `legacy_caps`.
  - `drawer-capture-legacy-existing.json` was dropped.
  - `legacy_caps` decoding is covered inline.
  - `aa233750` and `38dc0dee` fixed follow-on test failures.
  - Exact-SHA master CI is green on `46c5202b` (run 45857851813).
  - `GRAMMAR_REVISION` in bob-cli is still 33.
  - Epic note #13 still lists "Step 1 Mac fixture regen" as remaining. That line is
    stale.
- **Tray stroke.** bob-plugins `0d0a31f` raised the tray's `stroke-width` from 1.45 to
  2.06 in its 16×16 viewBox. The tray is masked `contain` into a 0.95em × 0.78em box.
  - Rendered tray stroke: 2.06/16 × 0.78em ≈ 0.1004em.
  - Rendered diamond stroke: 1.75/16 × 0.92em ≈ 0.1006em.
  - bob-cli `docs/task-drawers.md` "Obsidian rendering" states the 5% match.
  - The drawn tray only spans x 2–14 of 16 user units, which is about 0.59em across.
- **Deploy state of bob-ledger-tools** (deployed `styles.css`):
  - mac: has stroke 2.06, a clean deploy at `deee0f21`.
  - athena: has stroke 2.06, from another checkout's dirty build based on `6bd399f3`.
  - apollo: lacks it. It serves another agent's uncommitted build at `313a8ca`
    (2026-11-26T12:16Z), which predates `0d0a31f`.
- **Symbols.** `sase bead epic-symbols bob-cli-ph` prints no entries. bob-cli's justfile
  has no `symvision` recipe.

### Not ours: current bob-cli master is red

bob-cli master `3659edc0` comes from snooze phase `bob-cli-q5.8.1.1`, which is still in
progress. On that commit the lib-test target does not compile:
`src/native/snooze/cli.rs` around lines 1876–1912 uses `super::notify` and
`super::receipt`. Both CI `lint` and CI `test` fail on this.

Do not fix snooze code. If master is still broken when you verify, follow the fallback
in Step 3.

### Ground rules

- Open `bob-plugins` and `bob-mac-capture` only through `/sase_repo`, and read each
  repo's `AGENTS.md` first.
- Read reference memory `tailnet.md` before you SSH to athena or mac. mac is
  best-effort: it is offline when the lid is closed.
- Commits and notes name bead `bob-cli-ph`.
- Never run `just check-full`.
- Never pass `-F` to `bob plugins sync` outside the documented refusal procedure.

## Step 1: Tray geometry and e2e evidence (bob-plugins)

> [!decision] tray_width = widen Edit the two data-URI glyphs `--bob-drawer-glyph` and
> `--bob-drawer-glyph-empty` in `plugins/bob-ledger-tools/styles.css`, in the
> `body.bob-task-drawer-marks` block.
>
> - **viewBox.** Change it to about `0 0 19.5 16`, so the mask fills the 0.95em × 0.78em
>   `::before` box. The aspect ratio is 0.95/0.78 ≈ 1.218.
> - **Scale and stroke.** The scale stays 0.78em/16 per unit, so `stroke-width='2.06'`
>   still renders at about 0.100em. Keep 2.06 unless the box changes.
> - **Shape.** Redraw the same family: a beveled-bottom tray with an inner shelf line,
>   wide and shallow. Its drawn span, stroke included, should be about 0.9em: at least
>   0.85em and no wider than the 0.95em slot. Its height should be well under its width.
>   A reasonable starting path is `M1.55 4.2h16.4v6.1l-2 2H3.55l-2-2zM2.8 7.6h13.9`.
>   Tune it by eye against the screenshots.
> - **Empty variant.** The dashed empty variant uses the same path and dash pattern.
>
> Check every reading-view rule that draws the tray from `--bob-drawer-mask`, and make
> each one size its box the same way.

> [!decision] tray_width = keep Leave `styles.css` unchanged. The e2e work below still
> applies, but only the stroke-parity assertion is added. Do not assert the drawn width.

**Both branches.** Extend `scripts/obsidian-task-drawers-look-e2e.cjs`. Its fixture
sheet already holds a requirement row (`- [ ] First requirement`) beside the drawers. In
page JS, for every drawer row and for the requirement row's diamond pseudo-element:

1. Read the computed mask image.
2. URL-decode the data-URI SVG.
3. Parse it with `DOMParser`, then read `viewBox` and `stroke-width`.
4. Compute the contain scale: `min(boxW / vbW, boxH / vbH)`.
5. Compute the rendered stroke in em: `stroke-width × scale / fontSize`.
   - For the diamond, find the selector that applies `--bob-req-mask` (styles.css around
     line 7588).

Then assert the results:

- Each tray's rendered stroke is within 5% of the diamond's.
- The existing slot-width and slot-height assertions stay.
- Under `widen` only: measure the drawn span. Use `getBBox()` on the parsed path in an
  off-screen SVG, then multiply by the scale and add the stroke. Assert a span of at
  least 0.85em and at most the slot width.

Update `scripts/test-obsidian-task-drawers-e2e.cjs` if its source checks need the new
assertions. Then run `node scripts/obsidian-task-drawers-e2e.cjs --self-test`.

**Run the look e2e on athena.** Obsidian and `xvfb-run` exist on athena, not apollo.

1. On athena, make a detached worktree of its bob-plugins checkout at freshly fetched
   `origin/master`.
2. Apply your working-tree diff to it (`git diff | ssh athena 'git -C <wt> apply'`).
3. Run:
   `xvfb-run -a node --experimental-websocket scripts/obsidian-task-drawers-e2e.cjs -e /snap/bin/obsidian --only look -r <scratch>/report.json`.
4. The run writes `drawers-{light,dark}-{live,reading}.png` beside the report. Copy the
   report and the four PNGs back to a scratch dir on apollo, outside every repo.
5. Remove the athena worktree.

**Review the screenshots.** View each PNG. Confirm all of these:

- the titles are verbatim, with no `**`;
- the tray, checkbox, diamond, and dot are distinct in silhouette and in grayscale;
- the tray's stroke reads as the same weight as the diamond's.

Register the four PNGs and the report on the epic with `/sase_artifact`
(`sase artifact create -b bob-cli-ph`). Epic note #2 lists the diamond-only baselines,
if you want a side-by-side comparison.

## Step 2: Docs

- **bob-cli `docs/task-drawers.md`, "Obsidian rendering"** (around lines 131–139). Keep
  the stroke sentence: it matches the requirement diamond within 5%, both about 0.10em.
  Under `widen`, also say the tray is drawn about 0.9em across in its 0.95em slot. Then
  run `just docs-gen` and `just docs-check`.
- **bob-plugins.** `docs/bob-ledger-tools.md` around line 301 mentions drawer trays. If
  any bob-plugins doc states the tray's geometry, update it. Then run
  `npm run docs:check`.

## Step 3: Gates

**bob-plugins.** Run these:

- `npm run build`
- `npm test`. Treat failures on the known flake beads `bob-cli-km` and `bob-cli-io` as
  flakes.
- `npm run docs:check`
- `node --test` on the twin-copy suites `scripts/test-task-drawers-twin-copies.cjs`,
  `scripts/test-task-reqs-twin-copies.cjs`, and
  `scripts/test-req-close-twin-copies.cjs`.

If `build:check` flags a stale generated bundle that is not caused by your change, say
so in the commit body and let `npm run build` regenerate it from committed `src/`.

**bob-cli.** Run `just check` on current HEAD. The bar is zero drawer failures and zero
failures overall, apart from the fallback below. Record the pass and fail counts.

**Fallback.** Use this only if `just check` fails solely because another active bead
broke code outside drawer work (at plan time, the snooze compile error above). Do not
touch that code. Instead:

1. Run `cargo test -p bob-obsidian --test cli --no-fail-fast` on HEAD, which does not
   compile the lib test module.
2. In a detached `git worktree` at `8025ab3c`, run the lib target filtered to the
   drawer-owned modules:
   - `native::capture_complete`, `native::capture_drawers`,
     `native::capture::block_diff`
   - `native::capture::task_blocks`, `native::capture_clip`
   - `native::gcal::description`, `native::gkeep::render`, `native::gmail::body::inert`
   - `native::task_block_layout`, `native::task_children`, `native::task_drawer_rename`
   - `native::task_merge`, `native::task_reqs`, `native::task_tidy`
3. Remove the worktree.
4. Cite both runs, plus note #13's full-suite result.

If the owning bead's notes do not already record the breakage, add one
`DISCOVERED ISSUE:` note to it.

**bob-mac-capture.** Nothing to run unless something you changed is read by the Mac.
Normally nothing is, because the Mac draws SF Symbol `tray`. Re-confirm, read-only:

- no generated drawer fixture contains `legacy_caps`;
- the README provenance still names grammar 33;
- the latest exact-SHA master CI run that contains `6ea5b8f6` is green.

## Step 4: Deploy bob-ledger-tools

**apollo, both branches.** Run `bob plugins sync -p bob-ledger-tools` from your
bob-plugins checkout (`--repo <checkout> --no-pull`), as bob-plugins `AGENTS.md`
requires after changes.

The sync will refuse, because apollo serves another checkout's uncommitted build at
`313a8ca`. Follow `AGENTS.md` and `docs/develop/deployment.md`, "When sync refuses an
overwrite":

1. Run `git fetch`, then
   `git log --oneline HEAD..@{upstream} -- plugins/bob-ledger-tools`.
2. If that build has landed, follow the stash / ff / clean-sync / pop / build / re-sync
   recipe.
3. If its author is still running, wait for it to land. Do not force.
4. Ask Bryan only if the build is unlanded and abandoned.

Afterwards, confirm with `bob plugins list -n` and a grep of
`~/bob/.obsidian/plugins/bob-ledger-tools/styles.css` for the new glyph.

> [!decision] tray_width = keep athena and mac already serve stroke 2.06. Re-verify each
> with
> `ssh <host> "grep -c \"stroke-width='2.06'\" ~/bob/.obsidian/plugins/bob-ledger-tools/styles.css"`.
> Then go straight to Step 5 in this turn.

> [!decision] tray_width = widen athena and mac can only take the widened tray from
> pushed master, which your final declaration creates.
>
> **This turn.** Finish Steps 1–4 for apollo. Leave `bob-cli-ph` open. Before
> `/sase_final`, stage a follow-up with `/sase_monitor`. It runs a bounded wait, at most
> about 30 minutes, that exits 0 once `git fetch` plus
> `git show origin/master:plugins/bob-ledger-tools/styles.css` in the bob-plugins
> checkout contains the new path data. Then submit the final declaration: commit every
> touched repo and keep the bead (`bead_action keep`).
>
> **Follow-up turn.** On athena (`ssh athena`) and mac (`ssh mac`, with the PATH prefix
> shown in bob-cli `docs/vault-git-sync.md`), run
> `bob plugins sync -p bob-ledger-tools`. That pulls each host's own bob-plugins
> checkout. Apply the same refusal procedure on each host. Verify each deployed
> `styles.css` with grep.
>
> If mac is unreachable, or a host's refusal says to wait on a still-running author, do
> not force. Record which host still needs `bob plugins sync -p bob-ledger-tools` in the
> close note and in the final response. Then do Step 5 in that follow-up turn.

## Step 5: Close epic bob-cli-ph

Do this in the turn that finishes Step 4.

1. Re-run `sase bead epic-symbols bob-cli-ph`. At plan time it lists nothing. Resolve
   any entry that appears before you close.
2. Run `sase bead close bob-cli-ph --note "<verification>"`. Close the epic exactly
   once: do not also request a close through the final declaration. The note covers:
   - the `just check` counts (or the fallback runs and why);
   - Mac fixture provenance (`6ea5b8f6`, bob-cli `db5a70cc`, grammar 33) and green
     master CI run 45857851813. Say that note #13's "Step 1 remaining" was stale;
   - the tray and diamond stroke numbers, the drawn span under `widen`, and the
     screenshot and report artifact refs;
   - per-host deploy evidence;
   - a pointer to the LAND TRIAGE note #12.

   Never use `--force`. If the close is refused, investigate: every phase was closed at
   plan time.

3. In the plans sidecar, add `status: done` to the frontmatter of
   `202611/true_case_drawers.md`, the PLAN path that `sase bead read bob-cli-ph` shows.
4. Your final response must say that bob-cli has no `symvision` recipe.

## Acceptance

- `just check` in bob-cli passes, or, only when another bead has broken master, the Step
  3 fallback runs are green. None of the 80 drawer nodes fails.
- The tray's rendered stroke matches the diamond's within 5%, and the e2e asserts it.
  Under `widen`, the e2e also asserts a drawn span between 0.85em and 0.95em.
- Light and dark Live Preview and reading-view screenshots are registered on
  `bob-cli-ph`.
- bob-ledger-tools serves the final tray on apollo. Under `widen`, it also serves it on
  athena and mac, or the close note names each host still pending and why.
- `bob-cli-ph` is closed with a verification note, and `202611/true_case_drawers.md`
  says `status: done`.
