---
tier: tale
size: small
title: "Land bob-cli-9j: override context on an empty `=#` / `=3#`, then close the epic"
goal: "bob capture-complete reports the state-based `override` object on every
  `pomodoro_start_name` field while one session runs, including an empty `=#` or `=3#`.
  Bob Mac Capture fixtures, tests, and README match that real output. Epic bob-cli-9j is
  closed with its plan marked done.

  "
proposed_by: bbugyi200.apollo.bob-cli-9j.land
proposed_with: claude/opus
bead: bob-cli-9j
create_time: 2026-10-18 19:55:33
status: wip
---

- **PARENT:**
  [202610/eq_overrides_running_pomodoro.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/eq_overrides_running_pomodoro.md)
- **BEAD:**
  [bob-cli-9j](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9j/README.md)

# Tale: finish landing epic bob-cli-9j

This tale is the remaining work found by the bob-cli-9j land agent. Nothing resumes the
landing after this tale, so its last step closes the epic itself.

## Why

The epic plan (`plan:202610/eq_overrides_running_pomodoro.md`) made the
`capture-complete` `override` object state-based. It is present on a
`pomodoro_start_name` field when exactly one timed session runs and no earlier item or
earlier same-line chain token is a whole-item session operator. The plan also says: "The
swap picker status now appears for `=#` while a session runs."

Phase bob-cli-9j.1 implemented that: `=#` at cursor 2 returned `Some(true)`. Phase
bob-cli-9j.2 (commit b9788c45, retiring `==`) then added this guard in
`pomodoro_start_override_at` (`src/native/capture_language/completion.rs`):

```rust
let (suffix, name, _) = named_start_name_at(item, cursor)?;
// While the name picker may expose an empty insertion point after `#`,
// the override banner only applies once the draft names its destination.
if name.is_empty() {
    return None;
}
```

The comment confuses two things. The completion `override` is not the start-card banner;
the banner comes from the dry-run preview. The completion `override` drives Bob Mac
Capture's swap-picker status (`Pick a Pomodoro to take over CAPTURE's 0920–0945`), the
`Takes over 0920–0945` row text, and the running row's
`Already running — = restarts it; pick another Pomodoro to swap in` hint. The picker
opens on the empty `=#`, the moment the user types `#`. With the guard, that first view
shows the plain-start copy. The swap copy appears only after the first letter of a name.

The Mac phase (bob-cli-9j.4) worked around this. Its fixtures serve the override rows
only under `FAKE_BOB_OVERRIDE=1`. `pomodoro-start-override-complete.json` is really the
`=#bugs` cursor-2 list. Its README says "An empty `=#` reports no override until a name
is typed." bob-cli-9j.4 note 2 raised this as a proposed follow-up. The land agent
judged it epic-caused, so it is fixed here instead of being filed as a task.

Verified on the current tree, worked-example vault, `BOB_NOW=2026-10-09 09:32:00`:

- `bob capture-complete -c 2 -f json -- '=#'` → context `pomodoro_start_name`, no
  `override`. Expected:
  `{"keeps_ledger":true,"running":{"pomodoro_name":"CAPTURE","line":5,"time_range":"0920-0945"}}`.
- `-c 3 -- '=3#'` → no `override`. Expected: `keeps_ledger: false` with the same
  `running`.
- `-c 2 -- '=#bugs'` → `override` present (correct today).

Everything else in the epic was verified complete by the land agent. Follow-up triage is
already done and recorded on bob-cli-9j: memory task bob-cli-9l, a +1 on bob-cli-9i, a
note on bob-cli-87, and Mac CI task bob-cli-9m. Do not file anything new for those.

## Part A: bob-cli (this repo)

1. **`src/native/capture_language/completion.rs`, `pomodoro_start_override_at`.**
   - Delete the `name.is_empty()` guard and its two-line comment.
   - Change the binding to `let (suffix, _, _) = named_start_name_at(item, cursor)?;`.
   - The doc comment above the function is already correct; leave it.
   - Leave `named_start_name_at` and `pomodoro_start_name_field` unchanged.
   - A cursor before the name field (on `=`, on `<X>`, or on `#`) must still return
     `None`, and retired `==` tokens must still return `None`.
2. **Unit tests, `src/native/capture_language/tests/completion.rs`.** In
   `pomodoro_start_name_context_reports_ledger_behavior`:
   - `pomodoro_start_override_at("=#", 2)` becomes `Some(true)`.
   - Add `pomodoro_start_override_at("=3#", 3) == Some(false)`.
   - Keep `("=#", 1)`, `("=#bugs", 1)`, and `("=3#bugs", 2)` as `None`.
   - Reword the comment to say that a cursor before the name field has no context, while
     an empty name field already reports the ledger behavior.
   - `retired_doubled_start_has_no_name_completion_or_override_context` stays as is.
3. **`src/native/capture_complete/tests/output.rs`.** In
   `named_start_completion_carries_keeps_ledger_and_running`:
   - The empty `complete("=#")` now asserts an override object with `keeps_ledger` true
     and `running` naming CAPTURE (line 3, `0920-0945` in that fixture), not
     `is_none()`.
   - Add an empty `complete("=3#")` case that asserts `keeps_ledger` false.
   - Update the comment. The idle, missing-day, crowded, and retired `==#` assertions in
     the next test stay.
4. **`tests/cli/capture/complete_editor.rs`.** In
   `capture_complete_override_name_json_carries_keeps_ledger_and_running`, the empty
   `=#` case (`-c 2`) asserts:
   - `context == "pomodoro_start_name"`;
   - `override.keeps_ledger == true`;
   - `override.running` set like the `=#CAPTURE` case.

   Add an empty `=3#` (`-c 3`) case with `keeps_ledger == false`. The idle `=#`, the
   retired `==#`, and the missing-day cases keep their current expectations.

5. **`docs/capture.md`.** In the `capture-complete` paragraph that begins "A named-start
   name field keeps context `pomodoro_start_name` and its candidates when exactly one
   timed session runs", add one sentence:
   - the object is reported as soon as the name field exists, including an empty `=#` or
     `=3#`, so a picker can describe the swap before a name is typed.

   Run `rg -n 'empty `=#`|names its destination' docs README.md src` and remove any
   other claim that an empty name field omits `override`.

6. **Verify bob-cli.**
   - Run `just check`. Do not run `just check-full`.
   - If `just check` fails only on known clean-base failures, record the node IDs and
     confirm that each one also fails on a clean checkout of `origin/master`. Known
     clean-base failures include bob-cli-9i's Mermaid PDF tests when `pdftotext` is
     absent, and bob-cli-8q's parallel PTY test.
   - Rebuild bob (`cargo build`). Smoke it in a scratch vault built like
     `worked_vault()` in `tests/cli/capture/pomodoro_override.rs`. The settings come
     from `write_toggle_task_settings`; `day.md`, `bob.md`, and `sase.md` match that
     function. Run with `BOB_DAY_FILE=<vault>/day.md` and
     `BOB_NOW='2026-10-09 09:32:00'`:
     - `bob capture-complete -b <vault> -c 2 -f json -- '=#'` → `override` with
       `keeps_ledger: true` and running CAPTURE, line 5, `0920-0945`;
     - `-c 3 -- '=3#'` → `keeps_ledger: false`;
     - `-c 4 -- 'x =#'` → no `override`;
     - `-c 3 -- '==#'` → `context: null` and no `override`;
     - an idle ledger (`## Pomodoros\n\n- [ ] () — SASE\n`) with `-c 2 -- '=#'` → no
       `override`.

## Part B: Bob Mac Capture (linked repo)

Open it with `sase repo open bob-mac-capture -r "<why>"` and use only the printed path.
Read its `AGENTS.md` first. Keep the change to fixtures, `Tests/Fixtures/fake-bob`
comments and routes, test expectations, and `README.md`. No `Sources/` change is needed:
`pomodoroStartPickerStatus`, `startNameOpenSecondary`, and `startNameRunningHint`
already render the override object.

1. **Regenerate the worked-example completion fixtures from the Part A bob.** Use the
   Part A scratch vault and environment.
   - `Tests/Fixtures/pomodoro-start-override-complete.json` ←
     `bob capture-complete -b <vault> -c 2 -f json -- '=#'`. Its `replacement` becomes
     `{2,2}`; it used to be the `=#bugs` list with `{2,6}`.
   - `Tests/Fixtures/pomodoro-start-override-complete-fresh.json` ←
     `bob capture-complete -b <vault> -c 3 -f json -- '=3#'`.
   - Write each file as bob's single-line JSON, the way the existing fixtures are
     stored.
2. **`Tests/Fixtures/pomodoro-start-named-complete-running.json`** (the named-start
   vault with BUGS running `0840-0905` at line 7) must now carry the override that real
   bob emits for an empty `=#`.
   - Regenerate it from a scratch vault that reproduces every existing candidate
     byte-for-byte, including the `ref` hashes. Compare with `python3 -m json.tool` and
     `diff`; only the new top-level `override` key may differ.
   - If exact reproduction is impractical, confirm on a reconstructed vault with that
     ledger shape that bob emits
     `{"keeps_ledger":true,"running":{"pomodoro_name":"BUGS","line":7,"time_range":"0840-0905"}}`.
     Then append exactly that object as the last top-level key, and say so in the commit
     message.
3. **`Tests/Fixtures/fake-bob` completion routes** (around the `"=#"`, `"=#bugs"`,
   `"=3#plan"`, and `"=3#"` branches).
   - Rewrite the comments so they describe real bob: an empty `=#` / `=3#` carries the
     override while a session runs.
   - Point the `=#bugs` and `=3#plan` routes at fixtures that match their real drafts.
     One option is to generate `pomodoro-start-override-complete-named.json` (`=#bugs`,
     `-c 2`) and `-fresh-named.json` (`=3#plan`, `-c 3`) from the same vault.
   - Keep the existing env switches (`FAKE_BOB_OVERRIDE`, `FAKE_BOB_POMODORO_RUNNING`)
     so that no `Tests/BobMacCaptureTests` setup changes.
4. **Test expectations.**
   - `Tests/CaptureCoreTests/CompletionRowContentTests.swift`: rename
     `testPlainStartNameFixtureCarriesNoOverride` to describe the new contract, for
     example `testRunningStartNameFixtureCarriesOverride`. It asserts `keepsLedger` true
     and `running` BUGS, line 7, `0840-0905`. Keep the other override fixture tests
     passing; `candidates.count` stays 4.
   - `Tests/CaptureCoreTests/BobProcessClientTests.swift`
     (`testCaptureCompleteStartNameRunningVariantTrailsRunningRow`): add an assertion
     that `overrideInfo?.running?.pomodoroName == "BUGS"`.
   - `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`
     (`testNamedStartRunningVariantLeadsWithNextUpAndTrailsRunning`): the running `=#`
     status now reads `Pick a Pomodoro to take over BUGS's 0840–0905` (en dash), not the
     plain `Pick a Pomodoro to start, or type a new name`. Change only that string
     literal.
   - Search `Tests` for other expectations that an empty running `=#` shows plain-start
     copy (`rg -n 'type a new name' Tests`), and update only the ones that use a
     running-session fixture.
5. **`README.md`.** Near the swap-picker paragraph (search "An empty `=#` reports no
   override"):
   - replace that sentence with one saying the swap-picker status and rows appear as
     soon as `=#` (or `=3#`) is typed while a session runs;
   - drop any other claim that a name must be typed first.
6. **Verify Bob Mac Capture per its `AGENTS.md`.**
   - Run `just portable-check` on athena. This host has no Swift, so rsync a scratch
     copy without `.build` and `.git`.
   - Best effort on the Mac: compile a scratch copy, run `bundle.sh` and
     `just typecheck-tests`, then delete the copy.
   - If the Mac is offline (it was during landing), say in the final report that the
     macOS targets are unverified. The only `BobMacCaptureTests` edit is a string
     literal.
   - Do not wait for macOS CI. Its Test step cannot compile today because of the
     unrelated bob-cli-9m (`RefsNewestTests.zone`).

## Final step: close epic bob-cli-9j

Do this in the same turn as the code above. Never wait for this work's own commit, push,
or CI.

1. Run `sase bead epic-symbols bob-cli-9j`. It reported no entries during landing. If
   any appear, resolve each one: wire it up, privatize it, add a non-test pragma, or
   delete it, following the Symvision epic-whitelist policy. Re-key a Justfile line only
   when a still-open later bead needs that exemption.
2. Close the epic with a verification note, for example:

   ```bash
   sase bead close bob-cli-9j --note "Land verified all five phases against the plan and source: 5ec1390e, b9788c45, ec9009fd in bob-cli; 4c391eb, 12219ef in bob-mac-capture. Restart/swap, overdue, close_first, yellow warning, --refuse-override (Io, exit 1, pomodoro_override_refused, identical in dry-run), retired == teaching errors, prose and ==x rows, docs, Mac banner, orange Return, guard, and refusal re-preview all match. Fixed the epic-caused regression from b9788c45: capture-complete now reports override on an empty =# / =3# while one session runs (<list the bob test names and results>); Mac fixtures, fake-bob, tests, and README now match real bob. Integration: post-start commits 9ec332c8, 196af9b3, f4cdf0bd, 81103c2e, 930b3495 do not touch the override grammar; bob-plugins and memory hold no == grammar. Follow-ups: bob-cli-9l (decisions record), +1 on bob-cli-9i, note on bob-cli-87, bob-cli-9m (Mac CI compile). just check: <result>. portable-check: <result>. macOS targets: <verified or unverified>."
   ```

   Fill the placeholders with real results.
   - Never use `--force` just to make the close succeed.
   - If the close is rejected for leftover `--epic-symbol` entries, clean them up and
     close again.

3. bob-cli's `justfile` has no `symvision` recipe. Confirm with
   `just --list | grep -i symvision`. If a recipe exists by then, run `just symvision`
   and confirm the whitelist is clean.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file. That is the PLAN path printed by
   `sase bead read bob-cli-9j -r "Need the plan path to mark it done"`
   (`plan:202610/eq_overrides_running_pomodoro.md`). Change only that field.
5. bob-cli-9j has no `parent_bead`; confirm that in the same `sase bead read` output.
   Nothing further needs closing.
