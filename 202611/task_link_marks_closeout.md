---
tier: tale
title: Finish task link marks and close bob-cli-f5
goal:
  Repair task link eligibility and reduced motion, prove natural refresh and one swap,
  and close bob-cli-f5.
size: medium
proposed_by: bbugyi200.athena.bob-cli-f5.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-f5
status: done
---

- **PARENT:**
  [202611/task_link_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_link_marks.md)
- **BEAD:**
  [bob-cli-f5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-f5/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-f5.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-f5.land.md)
- **COMMITS:**
  - [f1ead7b](https://github.com/bobs-org/bob-cli/commit/f1ead7bf6499b69c03265af2d6a5e8265d647307)
    — docs(task-link-marks): add fence, comment, and managed-row vectors and liveness
    rules

# Finish task link marks and close bob-cli-f5

The landing audit found four reproducible display-contract defects and an e2e acceptance
gap. Repair only that remaining work, verify it, and finish the epic's landing in this
same coding turn. This tale has no separate land agent.

Read `sase bead read bob-cli-f5 -r "Need landing audit and final decisions"` and
`sase artifact read plan:202611/task_link_marks.md "Need the approved contract"`. The
existing phase beads bob-cli-f5.1 through bob-cli-f5.4 are closed. Their reported
implementation is present in bob-cli commits a1233eb8 and 4ea4c330 and bob-plugins
commits a67b8cd, 13d40ac, bedc7e2, and 32c9e03. Keep the settled decisions:
`ledger_rows = skip`, `dim_closed_links = no`. All changes remain display-only.

The durable reproduction and audit is `file:explicit:be7ba1208ebb2e69d16e7648`; read it
with `sase artifact read` if additional context is needed. Test evidence is
`file:explicit:3f28f3367c36f04929e7f6a0` (focused suite),
`file:explicit:e2d6e77a67e959ecb6b61bdb` and `file:explicit:59db89a083d8355365b792e0`
(isolated CPU budgets), and `file:explicit:bb831eb0037472bbaee6798e` (UTC control).

Use `/sase_repo` to open bob-plugins, read its AGENTS.md, and use only the returned
checkout. Edit source fragments, never generated `main.js`; register any new fragment
and keep each hand-edited fragment at or below 1,000 lines. The canonical contract is
bob-cli `docs/task-link-marks.md`. Read the docs reference memory before editing that
contract. No memory edits, Rust behavior changes, or new CLI options are needed.

## 1. Repair Markdown context and managed-row classification

In bob-plugins `plugins/bob-ledger-tools/src/148-task-link-marks.js`, replace the
independent, overly broad fence/comment accounting with correct context tracking for the
supported Markdown forms. Preserve the once-per-document model and viewport-limited
decoration build.

Reproductions on 32c9e03, using the existing exported core helpers:

- `taskLinkMarkLineEligible(["````markdown", "```", "[[bob#^fix]]", "````"], 2, "source.md")`
  returns true; it must return false. A closing fence must use the opening marker, have
  at least its length, and satisfy closing-line syntax. Cover tilde fences, shorter
  marker runs, and fence-like content as well.
- `taskLinkMarkLineEligible(["`%%` is a literal delimiter", "[[bob#^fix]]"], 1, "source.md")`
  returns false; it must return true. Comment delimiters inside inline code or fenced
  code cannot change later prose eligibility. Track real multiline comments without
  suppressing valid text following their closing delimiter; span-level exclusions should
  agree with document context.

In `285-plugin-task-link-marks-reading.js`, `taskLinkMarkReadingOwnRowIsEdge` wrongly
treats any list item beginning with `Links` as a managed row. The existing Reading test
harness reproduces this: render a normal `ul > li` containing `Links worth reading: `
and an internal `bob#^fix` anchor, with the resolver returning a known ready task. It
gets no ring; it must get one. Narrow recognition to actual managed-label
syntax/structure and the existing `mentionIsEdgeLine` contract. Also narrow the source
fallback `taskLinkMarkIsManagedRowText`, which currently suppresses arbitrary prose
containing `**LINKS**` anywhere. Keep malformed-but-recognized managed rows and actual
DEPENDS ON, BLOCKS, LINKS, LINKED FROM, and Sub-projects rows unmarked.

Add behavior regressions through both core helpers and the real decoration/DOM builders.
Include ordinary prose starting with managed-label words, actual managed rows,
code-contained comment markers, real comments, and code-fence boundaries. Add focused
conformance cases to `docs/task-link-marks.md` without changing the approved behavior or
existing vectors.

## 2. Make reduced motion actually disable swap animation

In `plugins/bob-ledger-tools/styles.css`, the animation selector
`body.bob-task-link-marks .bob-task-link-mark.bob-task-link-mark-swap` outranks the
reduced-motion selector `.bob-task-link-mark`. Chromium with
`--force-prefers-reduced-motion` and the production CSS reports
`matchMedia(...).matches === true` and computed
`animationName === "bob-task-link-mark-swap"` for that host.

Fix the cascade so computed animation and transitions are disabled under reduced motion
while normal-motion swaps remain intact. Add a browser/computed-style regression to the
existing visual/e2e tooling; checking that a media-query string exists in CSS is
insufficient. Use a short scratch TMPDIR for Chromium on athena to avoid its Unix-socket
path limit.

## 3. Prove natural cross-note refresh and one swap

Strengthen `scripts/obsidian-task-link-marks-e2e.cjs` and its offline self-test. Around
lines 918-945, the current test directly calls `plugin.runTaskLinkMarksRefresh(true)`
after polling the resolver. Remove that intervention from the acceptance path. Trigger
only the normal task edit in the isolated scratch vault, then observe DOM updates caused
by normal Tasks/metadata events. Verify both an already-open Live Preview source and an
already-open Reading source update their target ring within a bounded refresh interval.

The current `cross-note-swap-once` check is only a boolean `sawSwap`. Observe the
changed target ring's actual animation starts (or equivalent counted DOM evidence) and
assert exactly one per existing rendered instance. Also prove first paint, unchanged
refresh, scrolling, and rerender do not create extra swaps. Do not count an unrelated
already-Done ring as success. If stronger checks reveal a production liveness/animation
bug, fix it in the existing model/refresh/Reading fragments and add a focused
regression. Preserve read-only rendering and the no-selection-rebuild rule; do not add
file reads to render.

Run the real test with `xvfb-run -a` and an isolated scratch profile/vault; never point
the harness at the live vault. Retain the six states, ledger and managed-row exclusions,
hover/query-row coverage, toggle, and equal-line-height assertions. Capture light/dark
screenshots, inspect them, and preserve report/screenshots as audited SASE artifacts
rather than relying on temporary paths in the close note. If the host cannot run the
real application, report precisely which acceptance checks remain unrun; do not
manufacture a passing result.

## 4. Integrate drift, verify, and deploy

The lander reviewed non-epic bob-cli commits f4112320 (incomplete capture offers) and
5546ed1a (Next keep docs), plus bob-plugins 2c70724 (Schedule defaults) and fetched
origin/master 9c40ce2 (optional Next keep reasons). They preserve the ring contract and
need no direct adaptation. Recheck later commits, including the base branch, before
closing. Preserve their Rotten Log parsing, manifest descriptions, and generated build
changes when rebuilding the plugin.

The lander completed the unrelated phase-follow-up triage; the epic's landing note
records every outcome: f5.1-.4 note #1's date failures were corroborated on bob-cli-et,
their startup fake failures on bob-cli-eh, and their Keep timezone assertion became
small CI task bob-cli-f7. The two wikilink CPU-budget failures from f5.3 #1 and f5.4 #1
became large CI task bob-cli-f8 (investigation needed, three unchanged-tree failures,
including isolated runs). The additional intermittent cold-chunk miss is recorded with
f8's measurement evidence. No proposal was silently dropped. Do not duplicate those
tasks or treat them as f5 work. All task-link tests passed in the lander's focused
162-test run; seven unrelated tests failed. Known failures must remain precisely
identified if they recur.

In bob-plugins, run `npm run build`, the focused task-link and progress/managed-row
regressions, `npm test`, `npm run validate`, and `npm run docs:check`. Distinguish the
triaged baseline failures from any new failure. Fix any new failure caused by this tale.
Follow plugin version conventions for the repair, regenerate plugin docs if required,
and run `bob plugins sync` from the opened checkout. Confirm the deployed plugin still
exposes `toggle-task-link-marks`; do not overwrite another agent's protected deployment
blindly.

In bob-cli, run `just docs-check` and the file-change gate `just check` after the
contract edits. Do not run `just check-full`. Use `/sase_monitor` for long commands when
needed, carrying this exact closeout obligation into the continuation. Do not wait for
this tale's future commit SHA, push, or CI result: its host commits only after the
coding turn ends, and same-turn epic closeout is normal.

## 5. Close bob-cli-f5 in this turn

After repairs and verification are complete:

1. Reread bob-cli-f5 and its descendants/notes, confirm all required work is done, and
   include the already-recorded phase-follow-up outcomes in the close note.
2. Run `sase bead epic-symbols bob-cli-f5`. Resolve every listed exemption by wiring it,
   making it private, adding a justified non-test pragma, or deleting it per Symvision
   policy. Re-key only to a still-open later bead that genuinely needs it. The lander's
   audit found no entries, but recheck at close time.
3. Close normally with `sase bead close bob-cli-f5 --note "<verification>"`. The note
   must identify the repaired context/row/reduced-motion defects, natural-refresh and
   single-swap evidence, current drift review, passing gates or separately triaged
   baseline failures, deployed build, screenshot/report refs, and each phase proposal's
   outcome. If stale symbols block close, clean them and retry. If a named phase is
   incomplete, finish or reopen it. Never use `--force` to make a successful landing
   advance; forced cancellation or supersession requires a deliberate reason and the
   corresponding resolution.
4. Run `just symvision` if available after closing. Neither inspected checkout had a
   symvision recipe at audit time; record its absence if still unavailable.
5. Open the plans repository through `/sase_repo`, read the artifact through
   `sase artifact read plan:202611/task_link_marks.md "Need completed epic plan"`, and
   set `status: done` in the frontmatter of its `202611/task_link_marks.md` file (the
   PLAN shown by the bead). Include that sidecar edit in the final declaration.
6. Run `sase bead read bob-cli-f5 -r "Need the parent link"` once closed. The landing
   audit found no parent_bead, so finish normally if that remains true. If a parent has
   since been attached, follow the parent-phase versus parent-plan readiness rules in
   the land-agent prompt: close only a proven-complete parent phase; for plan ancestors
   reread all descendants/notes, linked plan and drift, retire their exemptions, close
   normally, verify symvision, and mark their plan done. Stop and note the blocker on
   the first incomplete or ambiguous ancestor.

Finish through `/sase_final` after all commands exit. Declare every edited repository,
including bob-plugins and plans, so the host commits all of this work.
