---
tier: tale
size: medium
title: Finish cached record routes and land bob-cli-l3
goal: Retain record targets in typed route completion, prove acceptance reaches the
  field menu, and finish the record-capture epic's landing.
proposed_by: bbugyi200.apollo.bob-cli-l3.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-l3
status: done
---

- **PARENT:**
  [202611/record_capture_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_capture_repairs.md)
- **BEAD:**
  [bob-cli-l3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-l3/README.md)

# Finish record capture and close bob-cli-l3

This is the remaining work from the land audit of **bob-cli-l3**, not a new feature or a
repeat of its three completed phases. One coding agent must implement this tale **and
finish the epic closeout below**. A tale has no land agent and no other agent will
resume this landing. The epic has no parent_bead.

Read `sase bead read bob-cli-l3 -r "Need landing audit and follow-up dispositions"` and
its linked `plan:202611/record_capture_repairs.md` using audited artifact reads. The
original plan's selected decisions remain final: `plus_commit_record_rows = skip`,
`album_refusal_presentation = refusal`, and `config_backed_album_test = repurpose`.
Preserve them. No memory changes, release version changes, or CHANGELOG edits are
needed.

## Evidence already established

- bob-cli phase l3.1 is commit `b7f74b58`: album-backed test fixtures and help snapshots
  only; the retirement-contract test remains. All migrated record tests pass in Linux
  exact-SHA CI 43322752069. The macOS job was cancelled only after lib and CLI results
  were printed; inspect those logs, not just its conclusion.
- Mac phase l3.2 is `e0c6c16`: route rows now use generic acceptance instead of Record
  Card acceptance, and typed +/, skips record rows. l3.3 is `9faa57b`: quiet bare-album
  refusal in submit/live/explicit preview, current-parse rewrite button, reset clears
  latestParse. Bob-served Return/Tab/click tests and all three bare-album tests pass in
  exact-SHA Mac run 43328622700.
- **Still broken:** `CapturePanelModel.cachedRouteCompletion` calls its `rankedTargets`,
  which delegates to `CaptureRefFileUnder.rankedTargets` in
  `Sources/CaptureCore/CaptureRefFileUnder.swift`. That helper calls `eligibleTargets`,
  retaining only inbox/area/project. It discards every record before
  `CaptureRecordRouteCache` can preserve its insertion and continuation. The filter was
  introduced by `6dddea35` before l3, but integrating it is part of l3's explicit
  cached-record acceptance scope.
- Run 43328622700 has seven failed assertions in l3.2's two tests:
  `testCachedRecordRouteAcceptsAndRequestsItsNextMissingField` gets `eat/ `, refuses
  acceptance and times out; `testRecordRouteRowsDoNotQuickCommitPlusOrComma` gets kind
  `inbox`. These remain epic work, despite l3.3 calling them pre-existing.
- Follow-up triage is recorded on l3 by the lander: duplicate gcal proposals from l3.1
  #1/#2 were corroborated on l2; docs-budget proposal l3.3 #1 is already repaired
  (plugin b6f3dd9: 5980 words, cap 6000; do not reopen jz); shell-deadline and
  pipe-drain failures have evidence on g8 / causal epic 3j and kl. The separate
  pre-existing macOS case-sensitive fixture failures are tracked by **bob-cli-l9**. Do
  not absorb those independent repairs. The detailed audit is
  `file:explicit:e7464a79500ce17eac54934f`; read it with `sase artifact read`.
- Integration review at primary b7f74b58 found no later base commits; since epic
  creation, Gmail transport 8e78584d, walker glossary 6981b04c, and gcal ack b6ed0e59 do
  not consume or conflict with record routes. Mac has only the two epic commits since
  creation. Recheck any newer drift before finishing.

## 1. Open and integrate

Work in the provided primary bob-cli checkout. Use `/sase_repo` to open
`bob-mac-capture`, and only use the printed linked checkout path. Read its AGENTS.md.
Open `plans` through `/sase_repo` for the final status edit. Fetch and integrate newer
base changes as necessary without discarding other work.

## 2. Retain record targets for typed routes

Separate general target ranking from File-under eligibility. Extract one shared ranking
implementation over the supplied targets, preserving the existing order: canonical
prefix, alias prefix, then other substring matches; input order within each bucket and
empty-query order remain stable. Typed route completion calls that general ranking with
Bob's targets. Keep `CaptureRefFileUnder.rankedTargets` as the restricted wrapper so
File under still excludes records, notebooks, and unknown kinds. Preserve its existing
restriction test and ordinary route behavior. Do not locally parse new capture grammar;
bob remains authoritative.

Keep `CaptureRecordRouteCache` insertion, label, recordType, and continuation through
the cached response. A selected movie route accepts to `@movies/ ` with the trailing
space and sends the next-missing-field request. Record Card contexts still use
acceptRecordCandidate. Do not make failed cached-route tests pass by forcing the server
path or deleting their cache assertions.

## 3. Prove the complete interaction with truthful fixtures

Repair the two failing CapturePanelModelTests to prove they use the cache: before
acceptance, assert the intended record target, exact replacement range, and no
capture-complete invocation. After explicit acceptance, assert the splice, caret, one
next-missing-field request, and the actual first missing-field value menu. For both +
and comma, assert the literal draft remains, no record accept or vault mutation occurs,
and the intended record list remains available as required by the skip decision.
Preserve Return, Tab, click, and ordinary-route coverage.

The current `FAKE_BOB_CAPTURE_COMPLETE_FIXTURE` overrides every response with a route
fixture, even the field continuation. Make fake-bob's record scenario respond by draft
and request mode so the follow-up returns the real record-value response.
Replace/regenerate `record-parse-movie-incomplete.json`, which currently has a fixed
`@movies/ ` input while also serving `Buy milk @movies/ ` and is not in
`Scripts/generate-record-fixtures.sh`. Use real Bob output with matching draft, title,
ranges, and cursor; extend the generator and record-fixtures-README rather than
inventing record JSON. Prefer a realistic title-bearing movie draft.

Extend the bare-album rewrite test in CaptureRecordPreviewPanelTests so clicking the
Bob-authored rewrite is followed by a fresh parse that actually enters record flow. Its
current fixed `FAKE_BOB_PARSE_FIXTURE` always returns the old bare-album parse and the
test only checks the splice. Keep the quiet live/explicit refusal, no generic
error/Retry, stale edit, and empty-reset assertions. Use bounded waits. Add focused
portable ranking tests for records and existing File-under restrictions.

## 4. Verify the current tree before closeout

Run appropriate focused tests, formatting, and **`just check`** for file-change
verification. Never run `just check-full`. In bob-cli, the existing l2, kl, g8, and l9's
case-sensitive fixture failures remain independent; record their exact outcomes and do
not report the entire suite or release train green when they fail. Prove the migrated
l3.1 nodes and retired-config regression pass.

For Mac Swift changes, obey AGENTS.md: `just portable-check` (use Athena if Swift is
absent locally), and best-effort Mac scratch-copy `bundle.sh` plus
`just typecheck-tests` for app/test targets; never use Bryan's checkout or install
there. Then run `Scripts/macos-ci-check` via `/sase_monitor` on the edited tree. This
gate snapshots the current files to a temporary precommit branch without requiring the
final landing commit. Require the two formerly failing tests, record interactions, and
bare-album tests to pass, with the whole Mac gate green. Do not waive l3 failures as
follow-ups. Follow the monitor skill and wait for its staging command to exit before
finalizing; its continuation must resume this tale through step 5. Preserve the
artifact/ref and exact tree evidence in bead notes. Run docs-check if any docs change.
The already-repaired word-budget report is not a reason to skip the check or refile it
without a fresh reproduction.

Do not arrange closeout after the final code's own commit SHA, push, or CI result. The
precommit tree gate is the verification step. Close the epic in the same turn as the
final verified code, then declare all changed repositories through `/sase_final`. If a
push is performed, follow the repository's exact-SHA monitor rule, but do not invent a
final-commit dependency that strands this tale.

## 5. Finish bob-cli-l3's landing in this tale

Re-read the epic, every child note, and linked plan; confirm no new descendants or
unresolved work and review any post-audit drift. Include the lander's full follow-up
dispositions from l3's note in the close note, including the separate macOS CI task ID
and the reason the old docs proposal was declined.

Run `sase bead epic-symbols bob-cli-l3`. It was empty in the audit. Resolve every listed
exemption (wire up, privatize, non-test pragma, or delete according to Symvision
policy), or re-key only to a still-open later bead that genuinely needs it. Inspect
relevant linked justfiles too. Then close normally:

`sase bead close bob-cli-l3 --note "<verified decisions, source/commit and drift review, cached and server route/field acceptance, bare-album refusal/rewrite, Rust fixture checks, Mac gate evidence, and all follow-up outcomes>"`

If it rejects stale symbols, finish cleanup and retry. If it names unfinished phases,
finish or reopen them; never force successful nested landing. Use force only for a
deliberate canceled/superseded outcome with an explicit reason, never to make this
success path proceed.

After close, run `just symvision` wherever available; if absent, record that explicitly
(it was absent from primary and Mac at audit time). Set `status: done` in the
frontmatter of **`plan:202611/record_capture_repairs.md`**, resolving its current PLAN
path with `sase bead read` and editing through the opened plans repo. Re-read l3's
parent link: there was no parent, so no ancestor closes are needed. Finally use
`/sase_final` with commit decisions for every changed repository.
