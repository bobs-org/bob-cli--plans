---
tier: tale
title: Finish capture alias typing-tip landing
goal:
  Repair client tip lifetime and adoption accounting, verify the integrated feature, and
  close bob-cli-j9.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j9.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j9
create_time: 2026-11-09 16:29:54
status: wip
---

- **PARENT:**
  [202611/capture_alias_typing_tips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_alias_typing_tips.md)
- **BEAD:**
  [bob-cli-j9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j9/README.md)

# Finish the capture alias typing-tip landing

Repair the remaining client state bugs in epic **bob-cli-j9**, verify the integrated
feature, and close the epic in this same implementation workflow. This tale has no
separate land agent. Its coder owns the final closeout below.

## Context and boundaries

Read `sase bead read bob-cli-j9 -r "Need landing audit and follow-up dispositions"` and
the latest landing notes. Read the approved epic through
`sase artifact read plan:202611/capture_alias_typing_tips.md "Need the typing-tip contract"`.
Open `bob-mac-capture` and `plans` through `/sase_repo` and use only the returned paths.
Read the Mac repository's `AGENTS.md` before editing it.

The audited baseline is bob-cli `d7906813` and bob-mac-capture `a65fb6a`. All four
original phases are closed. Rust commit `4b59ae81` stamps forward aliases and proves
reverse tips through the complete rewrite pipeline. Mac commits `bd06c21`, `3131414`,
`0eeba70`, and `e5833d6` supply the models, policy, list, and picker hosts. Preserve the
accepted decisions: the picker tip belongs at the card foot; only glossary:capture-alias
was authorized and that sentence has landed; no further memory changes are needed. Keep
schema version 1 and existing grammar.

The independent record-preview failures were repaired by `2f1d6dd`, and the `.addTo`
compile gap was repaired by `3131414` with copy refined in `b1a90fa`. The modifier
escape work in `ef5dc4a`/`b1a90fa` is already integrated; retain it. The lander's note
records dispositions for every original PROPOSED FOLLOW-UP. The future three-week
adoption review is separate from implementing this tale.

## 1. Do not mistake a changed response for an old Bob

In `Sources/BobMacCapture/CapturePanelModel.swift`, `applyAliasTipRewrite` currently
checks `reportsSigilAliasTips` before checking whether the response changed the draft.
The wire contract intentionally omits `sigil_alias_tip` on **every changed response**,
including modern Bob. For example, the current Rust source rewrites `@cash:task^` to
`@cash^task` with `rule: switch_block_id_separator`. A physically typed caret starts the
independent tip lane as well as that normal rewrite lane. Its changed reply currently
marks this process client's identity tip-incapable, suppressing all future tips until
that identity changes.

Ignore changed responses before capability detection or learning-state mutation. Never
apply edits from the tip lane. Only an otherwise valid unchanged cursor response with
the capability field absent proves old-Bob incompatibility. Retain explicit-null
ineligible counts and the existing identity reset behavior.

Add a model regression that supplies a changed separator-toggle response with no tip
field, then a valid route-tip response for a later physical key on the same client. The
later request must still run and the usual first-frame rules apply. Keep the existing
test proving that a genuinely old unchanged reply disables the lane. Include a changed
response racing the main rewrite without altering the draft or Undo behavior.

## 2. Bind admission to the original surface opening

`applyAliasTipRewrite` currently rejects a late reply only while `completionVisible` or
`pickerVisible` is true. `dismissCompletion` drops an already pending tip, but does not
retire an in-flight request. Once a list opens and is dismissed before the reply
returns, the same draft prefix can still validate and a later reopen can show that
obsolete tip. `dispatchCapture` also invalidates the rewrite lane, but does not retire
the tip lane or clear its presented row.

Give each physical tip request a bounded lifetime that remembers whether its first
surface has already opened, even after that surface closes. At the existing
cached-route, server-completion, and picker admission points, consume that opening
opportunity for both pending and in-flight replies. A late answer may contribute one
eligible miss, but can never become pending for a second opening or migrate between list
and picker. Do not cancel a valid early reply merely because a picker's setup internally
clears the previous completion state.

Retire display eligibility on a real submit, accept, hide, and draft replacement; clear
the presented tip when its host closes. A valid shown tip stays unchanged while its
original host filters. Keep every tip request off the completion, preview, Similar
Tasks, and Return critical paths. Recheck session eligibility at admission so an
intervening escape or another shown tip cannot be bypassed.

Use controlled reply ordering with bounded waits in model tests: reply before open;
reply while open; open then dismiss then reply then reopen; list-to-picker; submit
before reply; filtering after admission; and hide/show with an old reply. Assert at most
one miss per chance, no late presentation, and unchanged draft, selection, candidate
choice, and Undo behavior. These tests should exercise the real opening methods, rather
than only installing a pending tip directly.

## 3. Count surviving aliases accurately

`Sources/CaptureCore/CaptureAliasTip.swift` increments weekly `hits` in
`AliasTipLedger.recordHit`, but `withdrawHit` removes only the recent-window hit. An
immediate Undo or literal escape therefore leaves a successful adoption in the weekly
measurement used by the approved stop rule. Correct both representations when the same
credited alias is withdrawn, using its original ISO week. An escape after a week
boundary must not decrement the new week's unrelated hits; missing or pruned buckets
must not become negative or be recreated merely for rollback. Keep withdrawal
idempotent, preserve JSON-string storage compatibility and the 12-week cap, and persist
no draft text or keystroke history.

Audit the in-memory `lastAliasHit` lifecycle too: `reconcileAliasTipHit` currently
forgets the credit after any edit other than the immediate pre/post draft. Ensure native
Undo that actually undoes the alias after undoing later typing withdraws that alias
once, without subtracting another gesture's hit. Use native undo/gesture identity or
bounded session bookkeeping; do not infer adopted aliases from prose.

Extend pure fake-clock ledger tests with same-week withdrawal, week rollover, repeated
withdrawal, and pruned history. Extend the existing native-editor tests to assert weekly
counts as well as `recent`. Exercise surviving hits and all supported escape paths,
including the integrated modifier `.`/`;` fast doubles, Space/Escape, relation `==`,
hash literal escape, and native Undo. Include picker lessons as well as route/hash, with
tips both on and off. Real forward-alias fixtures must include Bob's `alias` object so
those tests exercise accounting.

## 4. Finish the deferred JSON reference integration

Read `sase memory read docs.md -r "Finish j9 JSON reference documentation"`. The
optional `docs/reference/json-outputs.md` entry deferred by j9.1 is now possible: the
notebooks change `d9066754` reduced the page to 5931 words. Add a concise mention of
`alias` and object/null `sigil_alias_tip` to the `capture-rewrite` row, linking to the
existing detailed contract in `docs/capture.md`. Do not duplicate the lesson table.
Recheck the live budget before editing and keep the page below its cap.

Mac docs `capture-panel.md` and `keyboard.md` were trimmed by `a65fb6a` to 5906 and 5981
words; the old three-page budget failures are resolved at this baseline. Update
`docs/typing-tips.md` only if necessary to describe corrected accounting.

## 5. Verify before closeout, without waiting for this tale's own commit

Inspect commits on both repositories' base branches since the audited baselines and
integrate any intervening changes affecting these paths. Keep the original proof rules,
preferences, visual slot, and local-only storage intact.

Run `just check` in bob-cli; never run `just check-full`. Probe a freshly built checkout
binary for the object, explicit null, absent changed-response field, and forward alias
stamp. The lander's existing `target/debug/bob` was stale and is not verification
evidence for these additive fields.

Run `just portable-check` in the Mac repository (on athena if Swift is unavailable
locally, following the tailnet memory). Compile the AppKit/SwiftUI changes: follow
`AGENTS.md` for a scratch Mac bundle and `just typecheck-tests` when reachable, and run
`Scripts/macos-ci-check` as the exact-tree **precommit snapshot** gate. Use
`/sase_monitor` for long checks, and retain these closeout instructions in its
continuation. Read the final result of each gate. The known double-equals flake is
`bob-cli-jb`; any newly caused failure remains this tale's work. Do not claim green from
the old picker snapshot: run 42696688042 passed lint/build and all 15 alias-tip panel
tests, but failed that known flake and skipped bundle/install. Require a green snapshot
gate for the repaired Swift. No step requires this tale's final commit SHA, its push, or
a CI run triggered by that future commit.

Run `just docs-check` in both repositories with the local linked checkout mounted using
their documented environment settings, plus `git diff --check`. Verify new tests fail
for the repaired conditions on the old implementation. Keep unrelated test
infrastructure work with its task; a `just check-full` failure after a passing
`just check` is not remaining epic work.

## 6. Close bob-cli-j9 and mark its original plan done

This is the final step of the tale, not work for a future land agent. Re-read
bob-cli-j9, all four original children, any new descendants, and their notes. Confirm
the landing issues above and every latest note are addressed. The original plan is
`plan:202611/capture_alias_typing_tips.md`, at `202611/capture_alias_typing_tips.md`
within the repository returned by `sase repo open plans`. The audited epic has **no
parent bead**.

Run `sase bead epic-symbols bob-cli-j9`. Resolve each entry by wiring, privatizing, a
valid non-test pragma, or removal per Symvision policy; re-key only to an actually open
later bead that still needs the exemption. Recheck descendants and linked-plan
readiness. If this tale has an assigned child plan bead, close that completed bead
normally first so it cannot block the parent close. Mark its own linked plan done as
required by the host. Do not force a successful nested landing.

Close the epic normally:

```sh
sase bead close bob-cli-j9 --note "Verified all four original phases, every landing and follow-up disposition, and intervening Rust/Mac commits; repaired tip capability detection, first-surface lifetime, and surviving-hit accounting; completed the JSON reference entry; verified the final tree with the recorded checks."
```

Include concrete test evidence and the original follow-up outcomes from the lander's
note in the actual close note. If symbol cleanup or unfinished descendants reject the
close, finish them and retry. Never use `--force` merely to succeed or to advance this
successful nested landing; canceled/superseded work requires a deliberate documented
reason and non-done resolution.

After closing, run `just symvision` in repositories that offer it; report recipe absence
otherwise (neither audited baseline offers it). Set `status: done` in the frontmatter of
**the original epic plan** `202611/capture_alias_typing_tips.md`. Read
`sase bead read bob-cli-j9 -r "Need the parent link"` again: no parent is expected, so
finish normally. If the relationship has changed, follow the supplied landing rules:
close only a completed parent phase, or fully re-audit a directly parented plan before
closing it; leave a blocker note at any incomplete ancestor. Finish with `/sase_final`,
declaring every changed repository for commit. All these closes and plan status updates
happen before the host commits this tale's code.
