---
tier: tale
title: Finish dot-route gesture invalidation and land bob-cli-iu
goal:
  Dot escape obeys the no-intervening-key rule, mounted docs pass their budgets, and
  bob-cli-iu closes with verified integration and follow-up triage.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-iu.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-iu
create_time: 2026-11-08 18:14:51
status: wip
---

- **PARENT:**
  [202611/dot_route_alias.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/dot_route_alias.md)
- **BEAD:**
  [bob-cli-iu](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-iu/README.md)

# Finish the dot-route escape and land bob-cli-iu

Complete the remaining work in epic **bob-cli-iu**, then close that epic in the same
coding turn. This tale has no land agent. Do not stop after implementing the repairs or
wait for another lander. The original plan is `plan:202611/dot_route_alias.md`; its
resolved path is printed by `sase bead read bob-cli-iu`.

The approved decisions stay **leading_dot = after_text**, **escape_speed = fast** (350
ms), and **restore_keys = restore**. No new decision, grammar revision, package version,
changelog, or memory edit is needed.

## What the landing review already verified

- Read both closed children, bob-cli-iu.1 and bob-cli-iu.2, every child note, the
  approved plan, and the actual code and commits: bob-cli `0b8b913f` and bob-mac-capture
  `e7ddfc6`. The Rust fallback preserves existing dot-to-caret and move-bullet
  precedence; it requires space/tab after body text, excludes a bare checkbox, unclosed
  backticks and wikilinks, and delegates eligibility to Bob's route completion. The
  response is the existing `expand_sigil_alias` rule with one byte replaced by `@`,
  unchanged cursor, and `Typed @ for .`. Parse/submit grammar stays unchanged.
- The Mac uses Bob's reply to arm the generalized equals gesture for `.`/`@`, keeps
  `.`/`^` unarmed, and implements the after-reply fast escape, pre-reply probe,
  Space/Escape restores, native Undo, and older-Bob fallback. Snapshot macOS run
  **42531359678** passed Lint/Build and all **17 CaptureDotRouteAliasTests**.
- At audit time primary HEAD and fetched origin/master were `5ef66125`; the only
  non-epic commit after `0b8b913f` was `5ef66125`, Gmail watched-target anchoring and
  recovery. Its files/callers do not intersect capture alias handling. Mac HEAD and
  origin/master were `e7ddfc6`, with no later commits. Existing x/s forwarding and
  type-ahead work are already in that commit's base. Recheck drift before editing,
  particularly the active bob-cli-is.3 repair in the same Mac controller/model.
- `sase bead epic-symbols bob-cli-iu` returned no entries. The primary justfile had no
  symvision recipe. The epic has **no parent_bead**. Recheck these at closeout.
- No fresh `just check` was run by the lander. Phase-reported Rust checks are historical
  evidence; the coder must run the current gate below.

## Follow-up triage is complete; preserve these outcomes

All proposals came from bob-cli-iu.2; its note #1 (`--list`) has no actionable scope.

1. Note #2, stale docs mount collision: duplicate **bob-cli-it**. The lander recorded +1
   after invoking production `validate_mount_targets` on a temporary stale index with
   empty tracked/manifest-owned sets and ignored-path classification. It still falsely
   calls the index a committed landing page. Do not create another task or repair that
   tooling in this tale. If stale generated output blocks docs-check, isolate the build
   or move aside only verified untracked, ignored mount residue; preserve authored
   files.
2. Notes #3 and #5 are one CI report and its tracker-selection guidance. Independent
   job-log comparison found **16 failing methods across three suites**, not 16 separator
   methods: 14 separator tests, one model plus-accept test, and the agenda
   clear-during-preview test. All 16 fail in base run **42523594058** (`2a39c2f`), which
   has 19 failing methods. The new eager `captureRewrite("xx")` capability probe adds
   calls and changes readiness; **bob-cli-is.3 notes #2–4** already identify and repair
   these lifecycle, assertion-count, comma-parse and agenda failures. The lander
   recorded the full evidence as a `DISCOVERED ISSUE:` on **bob-cli-is**, the causal
   active epic. No new task, no +1 to bob-cli-h7, and no reopening bob-cli-f3: their
   earlier alias/picker failure sets are different.
3. The Mac documentation overages below are caused by bob-cli-iu itself, confirmed
   independently after bob-cli-ir.2.1's lander reported them on this epic. They remain
   epic work, not a new follow-up or a corroboration of the older protocol overage in
   bob-cli-h9. That older protocol state was already below the cap at `2a39c2f`.

CI evidence: https://github.com/bobs-org/bob-mac-capture/actions/runs/42531359678 and
https://github.com/bobs-org/bob-mac-capture/actions/runs/42523594058.

## 1. Make dot-escape invalidation cover the whole gesture

Open `bob-mac-capture` with `/sase_repo` and use only the returned checkout. Read its
AGENTS.md. Work in `Sources/BobMacCapture/CapturePanelController.swift`,
`CapturePanelModel.swift`, and `CaptureKeyCommandRouter.swift` as needed. Keep one
shared equals/dot mechanism, with Bob as the sole grammar authority.

Two source-level defects remain at `e7ddfc6`; the lander did not execute AppKit
reproductions locally:

- Around controller line 2914, `isDotFastSecond` only checks timing eligibility and
  `event.characters == "."`. Command/Control/Option vetoes appear in the router but not
  this arm-preservation test. A modified dot is refused as an undo yet leaves the old
  dot arm available for a subsequent unmodified dot or restore. Apply the same
  key/modifier/repeat/marked-text eligibility when deciding whether to preserve the dot
  arm as when actually claiming its undo. An intervening modified dot must invalidate it
  even when no text or caret changes.
- The clearing block runs only when `equalsGestureArm?.key == "."`, and
  `clearDotGestureArmForInterveningKey` also requires that applied arm. Before Bob
  replies, only `pendingEqualsFirstKey` exists. A nonmutating intervening navigation key
  can leave this pending dot eligible for a later fast-dot probe. A late first reply can
  also create an arm after the gesture has been interrupted. Track or consume pending
  dot eligibility at key-down as well as applied-arm eligibility, and ensure
  invalidation survives the asynchronous reply. Preserve the first dot's ordinary
  Bob-authorized rewrite while preventing stale escape/restore state from being created.
  Do not change the equals gesture's existing clearing policy.

Add regressions in `Tests/BobMacCaptureTests/CaptureDotRouteAliasTests.swift` through
the real controller with the hosted editor and fake Bob:

- Arm with first timestamp 1.0, route a Command/Control/Option dot at 1.1 without
  inserting text, then route a plain dot at 1.2: the second gesture must not collapse
  the alias. The modified intervening key must also disable subsequent Space/Escape
  restoration. Parameterize where practical.
- Delay Bob's reply with a deterministic bounded barrier if available (otherwise the
  existing delay knob and condition-based waits). Type the first dot, route
  Down/Up/Ctrl-N/Ctrl-P without changing the buffer/caret, then type the second dot
  within 350 ms before the reply. No double-dot collapse probe may claim it. Also
  release the first reply before the second dot in a companion case and confirm no
  interrupted escape/restore arm reappears.
- Keep genuine uninterrupted double-dot behavior before and after reply, exactly 350 ms
  versus 351 ms, auto-repeat/marked-text/zero-time guards, caret movement, Space/Escape
  restores, native Undo, old-Bob no-op, dot-to-caret, and all existing
  equals/hash/separator behavior passing. Do not weaken tests or add untimed waits.

## 2. Restore the mounted documentation budgets

Read `sase memory read docs.md -r "Repair dot-alias documentation budgets"` before
editing. At `e7ddfc6`, production `bob_docs.markdown.count_words` reports:

| Mac source page         | Before (`2a39c2f`) | After (`e7ddfc6`) |  Cap |
| ----------------------- | -----------------: | ----------------: | ---: |
| `docs/capture-panel.md` |               6000 |              6088 | 6000 |
| `docs/keyboard.md`      |               6000 |              6060 | 6000 |
| `docs/protocol.md`      |               5995 |              6014 | 6000 |

Condense these three source pages without dropping any binding behavior, existing
feature, compatibility detail, URL, or anchor. Retain the dot-to-caret precedence,
spaced dot route gate, literal at line/bullet starts, 350 ms escape, empty-query
Space/Escape restores, no-intervening-key rule, and old-Bob fallback. Clarify that the
double-dot escape is for a dot after body text: a true item-start `.bashrc` is already
literal under `after_text`. Keep each page below 6000 with modest headroom; do not raise
the cap or restructure published paths. Edit the Mac source docs, not the mounted copies
in bob-cli.

## 3. Verify the integrated candidate before closeout

- Refresh both repositories' upstream history and integrate any overlapping landed work,
  particularly bob-cli-is.3. Re-read relevant new bead notes; preserve its probe,
  type-ahead and readiness repairs. Do not duplicate that active epic's work.
- In bob-cli run **`just check`**, never `just check-full`. Run the exact wire smoke
  `capture-rewrite -c 10 -f json -- 'Buy milk .'` using this checkout's built binary and
  check the approved single-edit response. This is an additional alias; keep versions
  and grammar revision unchanged.
- In the Mac repo run `just format-lint` and `just portable-check` (on Athena if Swift
  is absent locally). Attempt the required scratch Mac `bundle.sh` and
  `just typecheck-tests`; do not use Bryan's live checkout. Run `Scripts/macos-ci-check`
  on the final candidate to compile/run macOS-only targets, including the new controller
  tests. Follow the repo's AGENTS.md. Use `/sase_monitor` for long commands and keep the
  continuation responsible for this tale's closeout. A local Mac outage is not
  permission to skip compiling; the precommit macOS CI gate can supply that evidence.
- Run **`just docs-check`** against the current Mac source mount (open any other mounted
  repo with `/sase_repo` before accessing it). Confirm all three pages pass the actual
  mounted reference budget. Preserve generated-output safety.
- Require the epic's behavior and regressions to pass. If an unrelated baseline failure
  remains, compare exact failing nodes with the recorded base and active owner, record
  its disposition truthfully, and avoid silently treating it as a new dot-alias
  requirement. A `check-full` infrastructure failure is not remaining epic work. Do not
  claim a green gate that failed.

## 4. Finish bob-cli-iu's landing in this coder's turn

This is the final step, performed once the candidate repairs and verification are
complete. **Do not order it after this tale's own finalizer commit, its SHA, push, or
post-push CI.** Precommit verification covers the candidate; closing the epic in the
same turn as its final code is the intended landing.

1. Read `sase bead read bob-cli-iu -r "Recheck epic closeout readiness and notes"` and
   each descendant, and audit the linked original plan and any intervening notes/drift.
   Preserve all follow-up dispositions above in the close note. If a new descendant is
   still open, finish it normally; never force a successful nested landing. Resolve an
   assigned tale bead normally first if it is a descendant that would otherwise prevent
   the epic close.
2. Run **`sase bead epic-symbols bob-cli-iu`**. Resolve every listed exemption by
   wiring, privatizing, adding a justified non-test pragma, or deleting it under
   Symvision policy. Re-key only to a still-open later bead that actually needs it. Do
   not leave any entry keyed to this epic or its closing phases.
3. Run **`sase bead close bob-cli-iu --note "<verification>"`** with a concrete account
   of both phase audits, repaired gesture invalidation and documentation, current
   verification results, upstream integration, and all triage outcomes. If exemptions
   cause rejection, fix them and retry. If unfinished named phases cause rejection,
   finish or reopen them; never use `--force` merely to make the close succeed. Force is
   only for a deliberate canceled/superseded outcome with an explicit reason, never a
   successful nested landing.
4. Run **`just symvision` if available**, recording absence if no recipe exists.
5. Open the plans repo with `/sase_repo`, audit-read the original plan via
   `sase artifact read plan:202611/dot_route_alias.md "Mark completed epic plan done"`,
   and set **`status: done`** in that plan's frontmatter. Include the plans repo in the
   finalizer declaration along with every other changed repo.
6. Re-read `sase bead read bob-cli-iu -r "Need the parent link"`. The verified link is
   absent, so no parent phase or plan needs closing. If the link unexpectedly changed,
   inspect it and apply the supplied landing rules rather than guessing. Finish through
   `/sase_final`, reporting the epic closed only after close succeeds.
