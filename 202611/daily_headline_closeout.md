---
tier: tale
title: Finish Daily Headline parsing and week views, then close bob-cli-mw
goal:
  Daily Headline honors its approved parsing and display contract, required validation
  is accounted for, and bob-cli-mw is closed with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-mw.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-mw
create_time: 2026-11-17 18:58:07
status: wip
---

- **PARENT:**
  [202611/daily_headline.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/daily_headline.md)
- **BEAD:**
  [bob-cli-mw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mw/README.md)

# Finish Daily Headline and close bob-cli-mw

## Outcome

Repair the concrete omissions found during the landing audit of
`plan:202611/daily_headline.md`, verify the integrated feature, and close epic
`bob-cli-mw` in this tale. This is remaining epic work, not a new feature. One coder can
complete it directly; there is no later land agent to finish this tale's closeout.

## Audit and constraints

The lander read the epic, every note on all four children, the approved plan, and the
actual source and commits across bob-cli, bob-plugins, bob-mac-capture, chezmoi, and the
vault. All four phases are closed, but the gaps below remain.

- bob-cli `2a58d06701cc74324c055970e69eaa9de8757dba` adds the shared parser and day
  headlines in week columns/children, brief day parts, and `capture-pomodoros --tasks`.
  Keep schema versions and brief/callout versions unchanged. Headline is null when
  absent with `--tasks`, omitted without it.
- bob-plugins `0d2fa4be42bdeb9f0f41cc490aea6624ce2574e6` adds table and tooltip text but
  omits headline hover titles, the two-line CSS clamp, and documentation. Its seven
  period-parts tests pass; they do not assert the missing behavior.
- bob-mac-capture `18de389ef75541ba76144470e5fb0426cccbf57b` implements the optional
  fixed headline row. Source review confirms same-day/stale handling, no-open
  visibility, measurement through the actual row view, two-line clamp, tooltip,
  VoiceOver label, and light/dark design-test fixtures. Its exact-SHA CI is run
  `44201688943`: lint/build had passed and Test was still running during the audit. No
  final macOS success is claimed yet. Later commit
  `394b3442df723dfdbc8362bba611e3d5d0585595` only changes the item-start dot route
  alias; its CI run is `44201825837`, also pending at audit.
- Vault commits `2659779dd` and `31395126c` correctly add the empty template slot and
  the staged POST row above the open Morning review, dropping the obsolete highlight
  clause only from that open occurrence. Later `cb5266405` changes a count, not these
  semantics. Chezmoi `aeaef838` and this host's applied config both have
  `post_stages: [periods, headline]`. Do not backfill daily notes or write a real
  headline for Bryan.
- Approved DECISIONS remain: `morning_clause=drop`, `stage_headline_row=stage`, glossary
  Daily Headline accepted and already added, decision-record rejected. No memory edit is
  needed or authorized by this tale. Daily headlines remain user-owned, read-only, inert
  to task state, without carry-forward, scores, or enforced length limits.
- Integration review: bob-cli `3914ab50` fixes the pre-existing JSON-docs budget;
  `9b8ab39c` adds dot aliases; `225a6fb0` adds panel glossary terms. The latter two do
  not conflict with the headline contract. Plugin Record Card and schema styling commits
  `c1da817c`/`b0678813` are likewise independent and must remain. Active epic
  `bob-cli-mx` is adding title-row walk marks; none had landed during this audit.
  Recheck subsequent drift so its snapshot/view changes preserve the headline field and
  fixed-row layout if they land before this tale completes.

Read the epic and its latest notes again, and read the original plan through
`sase artifact read`. Use `/sase_repo` to open linked repos and their instructions
before reading or editing them. Resolve all checkout paths at runtime; do not use
another agent's checkout. Read the docs reference memory before docs changes.

## 1. Complete the shared parser's existing contract

Work in `src/native/daily_headline.rs`, keeping `periods/note.rs` on the same
paragraph-ending helper and `periods/brief.rs` on the shared wiki-link helpers.

The lander compiled the actual `daily_headline.rs` and `markdown.rs` modules together
with `rustc --edition=2024` and observed these reproducible failures:

| Input                                                            | Current display          | Required display |
| ---------------------------------------------------------------- | ------------------------ | ---------------- |
| `**Headline:** Ship` followed immediately by `+ unrelated list`  | `Ship + unrelated list`  | `Ship`           |
| `**Headline:** Ship` followed immediately by `1) unrelated list` | `Ship 1) unrelated list` | `Ship`           |
| `**Headline:** Read [the guide](https://example.test/a_(b))`     | `Read the guide)`        | `Read the guide` |

Recognize the supported Markdown list markers at paragraph boundaries, including `+` and
ordered `)` forms, with space/tab separation; preserve normal prose and soft wrapping.
Flatten a valid inline link's entire destination, including balanced parentheses and
escapes, instead of stopping at its first `)`. Preserve the raw Markdown `text` field
and the existing explicit display-marker rules. Prefer an existing parser or one bounded
shared scanner over ad-hoc repeated replacements.

Remove the duplicate wiki-link scanning loop: `flatten_wikilinks` and private
`replace_wikilinks` currently repeat the same traversal. Share the replacement primitive
while retaining the brief wrapper's intentional removal of stray square brackets and the
Daily Headline path's need to retain Markdown-link syntax until it is flattened. Do not
broaden the brief's display contract accidentally.

Add focused regression tests for the failing inputs and surrounding text, malformed
links left readable, and the shared period-headline boundary. Retain all eight existing
Daily Headline tests. The lander ran those plus six markdown tests through an isolated
rustc test harness: 14/14 passed, demonstrating the current coverage gap.

## 2. Finish the week table and strip presentation

In bob-plugins, edit source fragments and rebuild generated `main.js`:

- `239-plugin-period-strip-parts.js`: every headline Summary cell gets `title`
  containing its full text, for both past/current and future rows. Preserve the future
  row's empty Worked/Done/Review cells and alignment. No-headline days keep the muted
  mostly fallback; empty future rows still stop after Child and Dates.
- `styles.css`: add `.bob-period-parts-summary-headline` wrapping and a two-line clamp
  (`white-space: normal`, supported line-clamp/box rules, overflow hidden). Keep normal
  headline ink for elapsed/current days and inherited faint ink for future days. The
  current stylesheet has no headline-specific selector at all.
- `239-plugin-period-strip-model.js`: honor the original precedence, a period row's
  `review.headline` when present, then its own day `headline`, then empty. Preserve
  week/quarter/year Review behavior and older-bob fallback.
- `239-plugin-period-strip-fence.js`: include column headline text explicitly in the
  paint signature. Child headline text already participates, so normal report refreshes
  are partially covered; test headline-only changes repaint the table and update strip
  hover text. Preserve cache timing and filtered event behavior.
- Add the omitted concise Daily Headline explanation to `docs/bob-ledger-tools.md`: Days
  Summary, future planned headline styling, two-line clamp/full-text hover, day-column
  tooltip, and older-bob fallback. Respect the page budget and source-owned mounted
  docs.

Extend `scripts/test-ledger-tools-period-parts.cjs` and appropriate existing fence tests
for full-text titles, future alignment, no-headline fallback, preserved period Review
headlines, and headline-only refresh. Inspect a narrow light/dark rendering to verify
the clamp and faint future text; do not rely solely on source-string CSS assertions.
Build and sync using this repo's documented deployment workflow. Follow the sync refusal
procedure; do not infer permission to force a deployment from the separate Record Card
epic's scoped approval.

## 3. Verify integration and finish the existing Mac gate

Recheck commits on each relevant repository since the audited hashes, including new
`bob-cli-mx` work if present. Integrate only concrete new conflicts or consumers of this
feature. Preserve Record Card and dot-alias work.

Run the meaningful parser, periods/brief, and capture-pomodoros headline tests, then
`just check` in bob-cli. Do not run `just check-full`. Run the plugin's build, tests,
and docs gates (`npm run build`, `npm test`, `npm run docs:check`) and bob-cli
`just docs-check` for the mounted documentation context. Use `/sase_monitor` for long
checks, with continuation instructions to finish this tale and the epic closeout; do not
terminate a provider turn with an ordinary subprocess still running.

The full Rust suite was not rerun in the landing audit. A bounded attempt at the catalog
test timed out while compiling dependencies, before executing any test; do not report
that timeout as a product failure. The source already documents three pre-existing
records-tidy failures, triaged below. If checks still report only those failures, retain
exact evidence, confirm baseline causality and current owner status, and report them
separately from repaired headline behavior. Any new failure caused by this epic must be
fixed before closing.

For Mac verification, inspect the exact-SHA runs above using GitHub Actions. If still
running, watch with `/sase_monitor` and carry forward the remaining closeout. Confirm
all required macOS steps, including Test, Bundle, signature, smoke, and
install/reinstall, not just Linux portable tests or Build. Inspect the new headline
light/dark render artifact when available. Check the later integrated commit's run as
well if it changes the tested state. Existing already-pushed commits satisfy this gate;
closing this tale must never depend on this tale's own future commit, SHA, push, or CI
run. No Swift edit is anticipated. If an actual headline Swift defect is exposed,
compile it per AGENTS.md, run portable-check and best-effort Mac scratch compilation,
and use `Scripts/macos-ci-check` on the worktree before closeout. Any after-push
observation is a separate host continuation, not a prerequisite for same-turn successful
epic closure.

## 4. Close bob-cli-mw in this tale

The lander completed `/sase_new_task` triage before proposing this tale and wrote the
outcomes on `bob-cli-mw`. Include them in the final close note:

1. `bob-cli-mw.1` proposed follow-up #1, JSON reference budget: declined as already
   resolved by `3914ab50`; canonical `count_words` at `225a6fb0` is 5256/6000, with both
   headline contracts retained. No task filed or reopened.
2. `bob-cli-mw.1` proposed follow-up #2, `records tidy` missing catalog/completion and
   canonical-path coverage: attached as `DISCOVERED ISSUE` to causal active epic
   `bob-cli-ms`. The lander reviewed the relevant source, `cfc3177c`, task-type/all-task
   searches, all 68 tasks from the last week, and 14 active epic scopes. No same-root
   task exists; old `4j`/`hp` reports have different resolved causes. No duplicate task
   filed. The note distinguishes the child's actual pre-headline reproduction from this
   audit's source corroboration.

Re-read `bob-cli-mw`, every descendant and new note, and its linked plan through audited
commands. Confirm the four original phase scopes and all remaining work above are
satisfied, and document test results plus any independently owned baseline failures. Do
not reopen completed phases merely as bookkeeping.

Run `sase bead epic-symbols bob-cli-mw`. None existed at audit, but inspect again.
Resolve every listed exemption by wiring it up, privatizing it, using an allowed
non-test pragma, or deleting it. Re-key only when a concrete still-open later bead needs
that exemption. Do not leave stale entries for another agent.

Then, in this implementation turn, run:

```sh
sase bead close bob-cli-mw --note "<concrete source/commit and integration verification, repairs, checks, Mac CI evidence, and both follow-up outcomes>"
```

If rejected, finish the named remaining work and retry normally. Never use `--force`
just to make closing succeed, or to advance a successful nested landing. Only deliberate
canceled/superseded work can use the explicit force/reason/resolution path, with an
actual justification.

After closure, run `just symvision` if available (there was no recipe at audit). Open
the plans sidecar via `/sase_repo`, resolve the PLAN path from the bead, and set
`status: done` in the frontmatter of `202611/daily_headline.md`
(`plan:202611/daily_headline.md`; normally `sase/repos/plans/202611/daily_headline.md`).
Do not hand-edit any bead status files.

Finally run `sase bead read bob-cli-mw -r "Need the parent link"`. The audited epic has
no `parent_bead`, so finish normally. If a parent was deliberately added since this
audit, inspect it: a phase parent is closed only after verifying its scope; a plan
parent requires its previous landing notes, all descendants/notes, linked plan
readiness, and post-child drift rechecked before normal closure, symbol cleanup,
symvision, and marking its plan done. Stop at an incomplete or ambiguous parent and
record its blocker rather than forcing it closed.

Finish through `/sase_final`, committing all repositories changed by this tale. The epic
close and plan status update belong in the same turn as the final repair code; do not
defer them until after that code's commit or push.
