---
tier: tale
size: medium
title:
  Land bob-cli-2p — plan-budget preview on named-start completion rows, help/docs fixes,
  and epic closeout
goal:
  Named-start completion rows that create a session (new and again) preview the plan
  budget in bob and show the cap badge in Bob Mac Capture. The named-start help and docs
  are accurate and covered by tests. Epic bob-cli-2p is closed and its plan is marked
  done.
proposed_by: bbugyi200.apollo.bob-cli-2p.land
bead: bob-cli-2p
status: done
---

- **PARENT:**
  [202609/named_pomodoro_start.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/named_pomodoro_start.md)
- **BEAD:**
  [bob-cli-2p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2p/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2p.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2p.land.md)
- **COMMITS:**
  - [02fe09e](https://github.com/bobs-org/bob-cli--plans/commit/02fe09e1e86153ee35804bd9c67dddde816b2c42)
    — docs(plans): mark named_pomodoro_start done

# Plan: finish landing epic bob-cli-2p (named Pomodoro starts `=<X>#pomodoro`)

## Context

Epic `bob-cli-2p` (plan `plan:202609/named_pomodoro_start.md`) shipped `=<X>#pomodoro`
named starts in `bob capture`, `bob capture-parse`, `bob capture-complete`, the docs,
and Bob Mac Capture. All five phases (`bob-cli-2p.1` … `bob-cli-2p.5`) are closed. The
land agent confirmed:

- Commits: `cba59ee`, `4a480bf`, `f41ab05`, and `8d79b1d` in bob-cli. Mac commit
  `219983f` in bob-mac-capture has a green macOS CI run, `36651453204`.
- Every worked-example row and the E1–E5, R3, R4, and W1 texts behave as planned.
- `cargo test` is green (1256 lib + 592 cli).
- `cargo fmt --check` is clean.

Clippy has one error: the pre-existing deny at `tests/cli/capture/pomodoro_name.rs:808`
(`|| true`). Epic `bob-cli-28` owns it; leave it alone. Every clippy warning predates
this epic.

Epic `bob-cli-2o` (plan budget) is still in progress and landed two commits during this
epic:

- `35b96b3` (bob-cli-2o.4) added the plan budget, strict mode, and the rule that
  `creates_pomodoro` completion rows preview `plan_themes_after` and `plan_themes_cap`.
- `754d1f3` (bob-cli-2o.5) added the `~<K>` drop.

Named starts already work with both:

- Strict mode never refuses `=#fresh`, and the budget meter and warning print.
- `=x~1 =#decks` chains correctly.

The remaining gaps below are this epic's work. Do only this work; do not re-plan or
re-implement the shipped feature.

### Gap A — the plan-budget preview is missing on "again" rows (bob-cli)

In `src/native/capture_complete.rs`,
`pomodoro_start_name_candidates_from_scan_with_hint` builds two kinds of
`creates_pomodoro: true` row:

- The **new** row gets `plan_themes_after` and `plan_themes_cap` through
  `pomodoro_start_creation_candidate` → `pomodoro_creation_candidate`.
- The **again** rows (completed sessions, `state: "completed"`) set
  `creates_pomodoro = true` but never set either field.

`plan_budget` counts only open entries, so starting an "again" session adds a theme. For
example, `=#plan` after a completed PLAN takes the plan from 3/3 to 4/3.

### Gap B — Bob Mac Capture never shows the cap badge on start-name rows

`Sources/CaptureCore/CompletionRowContent.swift` appends the red `"\(after)/\(cap)"`
badge only in `case .pomodoroName` create rows. The `case .pomodoroStartName` **New**
and **Again** branches ignore `planThemesAfter` and `planThemesCap`. The real-bob
fixture `Tests/Fixtures/pomodoro-start-named-complete-new.json` already carries
`plan_themes_after: 4, plan_themes_cap: 3`, but the row shows only `["New"]`.

### Gap C — help and docs defects

- `src/native/capture/cli.rs` (the whole-item start paragraph of `long_about`, around
  line 194) renders the typo ``=`<X>`#<pomodoro>``. It should read
  `` `=<X>#<pomodoro>` ``.
- The same paragraph (around line 204) says "The start needs a future `- [ ] ()`
  placeholder". That is true only for a bare `=`/`=<X>`, because a named start creates
  its session when needed.
- `docs/capture.md`, "Starting the next Pomodoro" guards paragraph (around line 834),
  says "`=` never creates an entry; that stays the job of `^route:block-id=`". That
  statement is now stale.
- `docs/capture.md`, "Plan budget and strict mode" (around lines 762–786):
  - The never-refused start list (`=`, `#NAME=`, and link starts) omits `=<X>#NAME`.
  - The completion sentence mentions only the `pomodoro_name` context.
- `docs/capture.md`, `bob capture-complete` action paragraph (around line 2682), still
  says `=x[<N>][!<M>]` and "dangling `,`/`!`". The CLI help already says
  `=x[<N>][!<M>][~<K>]` and "`,`/`!`/`~`".
- The execution phase called for a help smoke test, but none exists. No test asserts the
  named-start help text for `bob capture --help`, `bob capture-parse --help`, or
  `bob capture-complete --help`.

## Steps

### 1. bob-cli: plan-budget preview on again rows (Gap A)

- In `pomodoro_start_name_candidates_from_scan_with_hint`, set each again row's plan
  fields right after `candidate.name` and `candidate.creates_pomodoro = true`:

  ```rust
  (candidate.plan_themes_after, candidate.plan_themes_cap) =
      plan_themes_after_for(plan_hint.as_ref(), <candidate's canonical name>)
  ```

  Use the same canonical name the row displays; fall back to the slug when `name` is
  `None`.

- Only `creates_pomodoro` rows carry the fields. Start, name-it, and running rows keep
  omitting them. The `pomodoro_name` context output must stay byte-identical.
- Update the `plan_themes_after` doc comment on `PomodoroNameCandidate` if needed.
- Update the `capture-complete` `long_about`: after the Pomodoro-start-name sentence,
  say that its new and again rows preview the plan budget with the same
  `plan_themes_after`/`plan_themes_cap` fields.
- Unit tests in the `capture_complete.rs` test module:
  - Build a `PlanCreationHint`, or go through a scan plus hint helper.
  - Assert that on a ledger with a completed PLAN and three open themes, the again row
    reports `plan_themes_after == before + 1` and the cap.
  - Assert that the new row is unchanged.
  - Assert that start, name-it, and running rows have `None`.
- CLI tests in `tests/cli/capture/plan_budget.rs`, reusing its helpers:
  - `capture-complete` on the draft `=#`, with a ledger of three open themes plus one
    completed entry and a config at the default cap. The again row carries
    `plan_themes_after: 4` and `plan_themes_cap: 3`. No non-create row carries either
    field.
  - `capture-complete` on `=#fr`: the new row carries the fields.
  - Strict mode (`plan:\n  strict: true\n`), a whole-item named start `=#fresh` that
    adds a fourth theme:
    - It succeeds, with exit 0 and `ok: true`.
    - It reports `plan_budget.added_themes == ["FRESH"]` and a `plan_theme_cap_exceeded`
      warning.
    - `pomodoro_start.created_pomodoro` is `true`.

### 2. bob-cli: help fixes and help assertions (Gap C)

- `src/native/capture/cli.rs`:
  - Replace ``=`<X>`#<pomodoro>`` with `` `=<X>#<pomodoro>` ``.
  - Reword the placeholder sentence so a **bare** `=`/`=<X>` needs a future `- [ ] ()`
    placeholder. A named start creates its session when no open entry matches. Both
    forms refuse while a timed entry is running.
- Add help smoke assertions. Put them in `tests/cli/help.rs`, or next to the existing
  `pomodoro_start` help assertions in `tests/cli/capture/parse_pomodoro.rs` around line
  1591:
  - `bob capture --help` contains `=<X>#<pomodoro>`, `bob capture '=#deep-work'`,
    `bob capture '=3#bugs'`, and `bob capture '=x =#bugs'`. It does not contain
    ``=`<X>`#``.
  - `bob capture-parse --help` mentions `=<X>#` (named start or incomplete state).
  - `bob capture-complete --help` lists `pomodoro_start_name`.

### 3. bob-cli: docs (Gap C)

In `docs/capture.md`, and keep the Contents list in sync if any heading changes:

- **Guards paragraph** (around line 834): a bare `=` never creates an entry.
  `=<X>#<name>` creates and starts a named session (see "Starting a named Pomodoro"),
  and `^route:block-id=` starts a task's session.
- **Plan budget and strict mode**:
  - Add `=<X>#NAME` named starts to the never-refused start list.
  - Note that a named start that creates a theme still reports `plan_budget` and the cap
    warning.
  - Extend the completion sentence: in `pomodoro_start_name`, the new and again rows
    carry `plan_themes_after` and `plan_themes_cap` too.
- **`bob capture-complete` section**:
  - Correct the close form to `=x[<N>][!<M>][~<K>]` and the separators to `,`/`!`/`~`.
  - In the `pomodoro_start_name` paragraph, mention that new and again rows carry the
    plan-budget preview.
- **`README.md`**: change it only if a sentence contradicts the above. The existing
  "session starts are never refused" wording is fine.
- Verify each changed example against a debug build in a temp vault, using `BOB_DIR` or
  `-b`, `BOB_DAY_FILE`, `BOB_NOW`, and `BOB_CONFIG_FILE`.

### 4. bob-cli gates

Run `cargo fmt --check`, `cargo test`, and `cargo clippy --all-targets --all-features`.

- Clippy must report no new warning or error compared with the base tree. The only
  expected error is the pre-existing `tests/cli/capture/pomodoro_name.rs:808` deny,
  which `bob-cli-28` owns.
- There is no `just check` recipe in bob-cli.
- Do not run `just check-full`.

### 5. Bob Mac Capture: cap badge on New/Again start rows (Gap B)

- **Open the repo.** Use `/sase_repo` with `sase repo open bob-mac-capture`. If the
  linked checkout is missing (as on Linux hosts), use
  `sase repo open gh:bobs-org/bob-mac-capture`. Use only the printed path, and read its
  `AGENTS.md` if one exists.
- **Sync first.** Epic `bob-cli-2o` (phase `bob-cli-2o.12`) may be editing this repo
  concurrently. Pull or rebase onto `origin/master` before editing and again before each
  commit.
- **Row badge.** In `Sources/CaptureCore/CompletionRowContent.swift`,
  `case .pomodoroStartName`:
  - Append the same red `"\(after)/\(cap)"` badge to the **New** row (`["New", "4/3"]`)
    and the **Again** row (`["Again", "4/3"]`).
  - Show the badge only when `planThemesAfter > planThemesCap`, the rule the
    `.pomodoroName` create row uses. Prefer one small shared helper used by both
    contexts. Keep the `.pomodoroName` behavior unchanged.
  - Make sure the badge reaches the accessibility label, as it does for `.pomodoroName`
    (see `testCompletionCreateRowDecodesPlanThemes`).
- **Fixtures.** Build bob from the bob-cli checkout (`cargo build`). Then regenerate the
  real-bob `capture-complete` fixtures whose again rows now carry the plan fields:
  - `Tests/Fixtures/pomodoro-start-named-complete.json`
  - `Tests/Fixtures/pomodoro-start-named-complete-again.json`
  - any other `pomodoro-start-named-complete*.json` that contains an again row

  Regenerate them against the same worked-example ledger, `BOB_NOW`, and cursor the
  `fake-bob` cases expect, so only the new fields change. Leave the other fixtures
  untouched.

- **Tests** in `Tests/CaptureCoreTests/CompletionRowContentTests.swift`, or
  `CapturePlanBudgetPresentationTests.swift` under "Completion cap badge":
  - New row over cap → `["New", "4/3"]`.
  - Again row over cap → `["Again", "4/3"]`.
  - New and again rows within cap, or without the fields → no cap badge, which keeps the
    existing `testPomodoroStartNameNewRowCreatesAndStarts` and
    `testPomodoroStartNameAgainRow…` expectations valid.
  - A decode test from the regenerated fixture.
- **README.** Extend the cap-badge sentence to the `pomodoro_start_name` **New session**
  and **Again** rows.
- **Verify on macOS.** Swift is not available on Linux.
  - Try `ssh -o ConnectTimeout=8 mac true`. If it works, rsync the checkout (excluding
    `.build`) to a temp dir on `mac` and run `just format-lint build test` there.
  - Otherwise iterate on GitHub Actions. This plan explicitly instructs you to use
    `/sase_git_commit` for each CI iteration in bob-mac-capture, with conventional
    `feat(capture): …` or `fix(capture): …` subjects.
  - Find the run with `gh run list -L 3`. Wait in the foreground with
    `gh run watch <id> --exit-status` and a tool timeout of about 30 minutes, or hand
    the wait to `/sase_monitor`.
  - Read failures with
    `gh run view <id> --log-failed | grep -E " error: |error: -\[|failed \("`.
  - Done means one CI run with every step green. Record the run ID in the close note.
  - Never weaken or skip an assertion to go green.

### 6. Close out epic bob-cli-2p (final step)

Do this in the same turn as the code above. Do not wait for this tale's own bob-cli
commit, SHA, push, or CI; closing in the same turn is the normal landing.

1. Run `sase bead epic-symbols bob-cli-2p`. It currently lists none. If any entry
   appears:
   - Resolve it: wire the symbol up, privatize it, add a non-test pragma, or delete it
     per the Symvision epic-whitelist policy.
   - Re-key the Justfile line only when a still-open later bead needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-2p --note "<verification>"
   ```

   The `<verification>` note must summarize:
   - **Phase review.** The land agent reviewed all five phases and their notes and
     re-verified the shipped feature:
     - commits `cba59ee`, `4a480bf`, `f41ab05`, `8d79b1d`, and Mac `219983f` (CI
       `36651453204`)
     - every worked-example row plus the E1–E5, R3, R4, and W1 texts
     - `cargo test` and `cargo fmt --check` green; the only clippy error is the
       pre-existing `bob-cli-28` deny
   - **Integration.** Integration with `bob-cli-2o` commits `35b96b3` and `754d1f3`:
     - strict mode never refuses named starts
     - `=x~<K> =#name` chains
     - the gaps fixed here: the again-row plan preview, the Mac New/Again cap badge, the
       help typo and assertions, and the docs corrections, with the new Mac CI run ID
   - **Follow-up triage.** Already recorded in the epic's notes:
     - The pomodoro_name.rs:808 clippy deny was routed to active epic `bob-cli-28` as a
       corroboration note.
     - The link-form "again" alignment was declined pending Bryan's approval.
     - The other out-of-scope items are intentional non-goals.
   - **Tale gates.** This tale's own gate results.

   Never use `--force` to make the close succeed.

3. Run `just symvision` if the recipe exists (`just --list`). bob-cli currently has
   none; if it is missing, say so in the final response.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202609/named_pomodoro_start.md`. Its path is the PLAN path printed by
   `sase bead read bob-cli-2p -r "Need the plan path for closeout"`. It is `status: wip`
   today.
5. `bob-cli-2p` has no `parent_bead`, so nothing further needs closing.
