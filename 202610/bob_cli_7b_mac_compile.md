---
tier: tale
title: Repair the dependency-drop Mac compile and close bob-cli-7b
goal: "Rename the illegal Swift `guard` bindings and repair the three toggle-test string
  literals that keep bob-mac-capture from compiling, then close epic bob-cli-7b. The
  bob-cli feature is already on master and is not part of this change.

  "
size: small
proposed_by: bbugyi200.apollo.bob-cli-7b.land
bead: bob-cli-7b
status: done
---

- **PLAN:**
  [plan:202610/capture_link_drops_blocking_deps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_link_drops_blocking_deps.md)
- **PARENT:**
  [202610/capture_link_drops_blocking_deps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_link_drops_blocking_deps.md)
- **BEAD:**
  [bob-cli-7b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7b/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-7b.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-7b.land.md)
- **COMMITS:**
  - [e6a1b40](https://github.com/bobs-org/bob-cli--plans/commit/e6a1b40071b390412a4fa80bc51abb747e9d7d8b)
    — chore(plan): mark capture_link_drops_blocking_deps done

# Plan: Repair the dependency-drop Mac compile and close bob-cli-7b

## Context

Epic bob-cli-7b is in progress. All four phases are closed. The land agent verified the
bob-cli work and the post-start drift, recorded follow-up triage on the epic (the note
that begins `FOLLOW-UP TRIAGE`), and left this compile fix as the only remaining epic
work. Do not re-triage follow-ups. Do not edit bob-cli source, docs, or memory. Do not
run `just check-full`.

What already checks out, and must stay as it is:

- bob-cli `92515f1` (cli-core), `bdfc8c3` (cli-gestures), and `c80cbde` (publish) are on
  `origin/master`. `resolve_link_guard` is the only guard entry used by
  `pomodoro_link.rs`, `task_toggle.rs`, `ensure_next.rs`, and `pomodoro_close.rs`.
  `assess_link_target` returns `DropDependencies` only after the §13.5.1 gates, in
  order: `project_blocked`, `project_task`, `schedule`, then drop; the writer refuses a
  bad Depends-On line with `dependency_line`.
- `docs/task-dependencies.md` still has the reworded PDG1 and PDG18–PDG27.
  `decisions:capture-link-drops-blocking-deps` is accepted.
  `decisions:projects-block-from-prj-line` lists both
  `decisions:subprojects-inherit-project-blocking` and
  `decisions:capture-link-drops-blocking-deps` under `superseded_by`.
  `decisions:task-deps-are-depends-on-links` is `superseded-in-part` with that same
  record. Both bodies carry the one back-link sentence.
- Drift since `92515f1`, excluding this epic's commits: `f2f2fd0` (bob-cli-7d.1 Go To
  grammar) does not call the link guard; `2d1adfa` (bob-cli-7c.1 Successor Offers) left
  the drop contract in place. No bob-cli edit is required.
- `sase bead read bob-cli-7b` shows no parent bead. After the close, do not close or
  note any parent.
- `sase bead epic-symbols bob-cli-7b` already prints no entries. Re-run it anyway in the
  closeout. Do not add a `--epic-symbol` line.

## 1. Repair the Swift compile in bob-mac-capture

Open the linked repo with `/sase_repo` and edit only that checkout:

```sh
sase repo open bob-mac-capture -r "Fix the dependency-drop Swift compile errors from bob-cli-7b.3"
```

HEAD when this plan was written was `a86eddb` (`feat(capture): add Go To picker core`,
bob-cli-7d.3), one commit after the epic's `8f89423`. Keep the Go To picker changes. The
Go To CI run
[38414162582](https://github.com/bobs-org/bob-mac-capture/actions/runs/38414162582)
fails on the same diagnostics as the epic's run
[38408350959](https://github.com/bobs-org/bob-mac-capture/actions/runs/38408350959).
Those are the only errors in the failed log.

`guard` is a Swift keyword, so `guard let guard` and `let guard` do not bind a value.
`CaptureLinkGuard.detail` is `String?`; `promoted` is `[String]`. Rename the binding to
`linkGuard`. Do not backtick-escape `guard`.

In `Sources/CaptureCore/CaptureLinkGuardPresentation.swift`, the initializer must read:

```swift
guard let linkGuard = capture.linkGuard else {
    return nil
}
```

and the two assignments must read `linkGuard.detail ?? ""` and `linkGuard.promoted`.
Leave the second `guard isLinkGuardKind || ...` statement as a real `guard`.

In `Tests/CaptureCoreTests/CaptureModelTests.swift`,
`testCaptureCommandResponseDecodesLinkGuardWithDropRefused` must use
`let linkGuard = try XCTUnwrap(success.linkGuard)` and then `linkGuard.reasons`,
`linkGuard.promoted`, `linkGuard.dropRefused`, and `linkGuard.detail`.

In `Tests/CaptureCoreTests/CaptureTogglePresentationTests.swift`, three literals open
with `""",` so the comma sits on the delimiter line (`nextDirectionJSON` around line
512, and both non-empty arms of `ensureNextJSON` around lines 576 and 588). Swift
requires the opening `"""` to end its line and the closing `"""` to start its line.
Replace each fragment with a raw string so the splice stays valid JSON. Whitespace does
not matter; the tests decode it.

One dropped dependency (`nextDirectionJSON`, and `ensureNextJSON` when
`droppedCount == 1`):

```swift
#"
, "dropped_dependencies": [
  {
    "note_path": "job.md",
    "block_id": "prep",
    "link": "[[job#^prep]]",
    "text": "Prep work",
    "status_symbol": " ",
    "status_name": "Ready"
  }
]"#
```

Two dropped dependencies (`ensureNextJSON` default arm) is the same raw string with a
second object after the first:

```swift
{
  "note_path": "ops.md",
  "block_id": "budget",
  "link": "[[ops#^budget]]",
  "text": "Budget work",
  "status_symbol": " ",
  "status_name": "Ready"
}
```

Keep every other key, the `?` / `:` shape, and the interpolation
`\(pomodoroNameField)\(droppedField)`. After the edit, search the repo for `let guard`
and for `""",`. Both must be gone.

This Linux host has no Swift toolchain. Do not install one, and do not wait for macOS
CI, a commit SHA, or a push. The closeout below happens in this same turn.

## 2. Close epic bob-cli-7b

From the bob-cli workspace:

1. Run `sase bead epic-symbols bob-cli-7b`. There should be no entries. If any
   `--epic-symbol` line is keyed to bob-cli-7b or one of its phases, resolve it (wire it
   up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy) or, only when a still-open later bead still needs the
   exemption, re-key that Justfile line to that open bead. Do not leave the judgment.
   `sase bead close` refuses while any entry remains.
2. Close the epic. Do not pass `--force`. If the close is rejected because named phases
   were never completed, finish or reopen them, or record the outcome deliberately with
   `--force --reason ... --resolution canceled|superseded`. Never force merely to make
   the command succeed.

```sh
sase bead close bob-cli-7b --note "$(cat <<'EOF'
Verified bob-cli-7b on origin/master. Phases 7b.1–7b.4 are closed and their notes are addressed. resolve_link_guard is what pomodoro_link, task_toggle, ensure_next, and pomodoro_close call. assess_link_target drops only after project_blocked, project_task, and schedule, and the writer refuses a bad Depends-On line with dependency_line. PDG1 and PDG18–PDG27 are in docs/task-dependencies.md. decisions:capture-link-drops-blocking-deps is accepted; projects-block-from-prj-line and task-deps-are-depends-on-links are superseded-in-part with the back-link sentences. Drift since 92515f1 needs no bob-cli edit: f2f2fd0 does not call the link guard, and 2d1adfa left the drop contract in place. Follow-up triage is the FOLLOW-UP TRIAGE note: percent-listen stays bob-cli-6m, ETXTBSY stays bob-cli-5l, and the Mac compile failure was this epic's remaining work. That failure was guard used as a Swift identifier in CaptureLinkGuardPresentation.swift and CaptureModelTests.swift, plus three illegal multiline strings in CaptureTogglePresentationTests.swift. Those are renamed to linkGuard and rewritten as raw strings. epic-symbols had no entries. No parent bead.
EOF
)"
```

3. Run `just symvision` and confirm the whitelist is clean.
4. Open the plans sidecar and set `status: done` in the frontmatter of
   `202610/capture_link_drops_blocking_deps.md` (the PLAN path
   `plan:202610/capture_link_drops_blocking_deps.md`). Change only that field.

```sh
sase repo open plans -r "Mark the bob-cli-7b epic plan done after close"
```

Do not close any parent. The epic read has no `parent_bead`.
