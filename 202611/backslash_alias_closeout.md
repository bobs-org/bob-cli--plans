---
tier: tale
title: Finish backslash alias documentation and land bob-cli-jt
goal: Correct the remaining alias documentation, mark the contract break, and close
  bob-cli-jt with its plan done.
size: small
proposed_by: bbugyi200.athena.bob-cli-jt.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-jt
status: done
---

- **PARENT:**
  [202611/backslash_percent_alias.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backslash_percent_alias.md)
- **BEAD:**
  [bob-cli-jt](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-jt/README.md)

# Finish the backslash alias documentation and close bob-cli-jt

The alias implementation is complete. Finish the small documentation and release
annotation gaps below, then perform the epic's landing yourself in this same turn. This
tale has no land agent waiting behind it.

## Verified implementation and boundaries

- Read `sase bead read bob-cli-jt -r "Need the landing scope and audit notes"` and its
  two closed phases, `bob-cli-jt.1` and `bob-cli-jt.2`. The approved plan is
  `plan:202611/backslash_percent_alias.md`; read it with `sase artifact read`.
- The original epic has no parent bead. Its plan file resolves to
  `sase/repos/plans/202611/backslash_percent_alias.md` within the current checkout;
  resolve it again instead of assuming a workspace-specific absolute path.
- Rust commit `6ed27c1e` implements backslash to percent, removes every backslash hash
  fallback and global-collapse trigger, and adds `directive_menu` tips. Mac commit
  `df8045f` consumes percent replies, preserves older Bob's hash reply, supports
  slow/fast double-tap escapes, and adds the percent typing-tip lesson.
- All notes on both phases were reviewed by the lander. Read the epic's landing audit
  and follow-up triage notes for their dispositions and verification results. Do not
  duplicate those follow-up beads. Audit evidence is
  `file:explicit:5d919abc2555b8f911456244`; completed workspace verification is
  `file:explicit:a7ca20870803705cc70efee4`.
- The lander reviewed all intervening bob-cli commits through `4c1f7dde` and Mac
  event-note commits through `df8045f`. Event-note input uses its separate editor
  routing; it must not acquire capture aliases. No production integration conflict was
  found. Recheck any newer commits before closing.
- Keep the final reviewer decisions: `hash_fallback=drop` and
  `glossary_capture_alias=yes`. The glossary was correctly updated already. This tale
  needs no memory changes, new directives, Swift behavior changes, or plugin changes.

## Remaining work

1. Open `bob-mac-capture` with `/sase_repo`, read its `AGENTS.md`, and read the bob-cli
   docs reference memory through `/sase_memory_read`. In Mac `docs/capture-panel.md`
   (the alias paragraph) and `docs/keyboard.md` (the backslash or slash row), replace
   the incorrect claim that backslash stays literal "after a word" with "glued to a
   word". State the positive spaced example `Buy milk \` becoming `Buy milk %`, while
   keeping line-start, doubled-backslash, older-Bob hash compatibility, slash literals,
   and double-tap/Undo facts intact. Here the examples denote one typed backslash, not
   two keystrokes.

   Condense nearby duplicated prose to remove this epic's growth on those two
   already-over-budget pages. Measured with `bob_docs.markdown.count_words`, Mac commit
   `df8045f` grew capture-panel from 6036 to 6041 words and keyboard from 6061 to 6069
   words. Finish at or below 6036 and 6061 respectively (or a lower newer baseline),
   preserving behavior facts and published anchors. The remaining inherited overage
   belongs to `bob-cli-jz`; do not broaden this into a site restructure. Keep
   `docs/typing-tips.md`'s true positive "after a word" lesson phrase unchanged.

2. In bob-cli `docs/capture.md`, make the migration explicit in the sigil-alias
   contract: backslash no longer opens hash menus, including the old slash-literal
   positions such as `=\` and `[[cash\`; slash remains the hash alias where eligible.
   Preserve the existing positive percent example and completion-owned eligibility. This
   also supplies a primary-repository landing change whose finalizer commit must carry
   the missing breaking-contract marker: use a subject such as
   `fix(capture)!: finish the backslash directive alias migration` and explain the
   retired hash behavior in the body. Phase commit `6ed27c1e` omitted `!`, which the
   approved plan and `decisions:bob-cli-releases-ride-a-daily-train` require for this
   changed CLI/JSON contract. Do not rewrite published commits, edit versions, or edit
   CHANGELOG.md.

3. Verify the two word counts, inspect all final diffs, and run `just docs-check` with
   current siblings opened through `/sase_repo`. On the audited tree its only failures
   were pre-existing reference budgets: capture-panel 6041, previews 6088, keyboard
   6069, protocol 6149, navigation-hotkeys 6132, each against a 6000 cap. Show that this
   tale removes the epic's own additions without creating a new diagnostic. Do not
   silently treat inherited failures as a green result.

   Run file-change verification with `just check` (or `sase tool run check`); never run
   `just check-full`. Consult the recorded lander run before repeating unrelated
   diagnostics. Lander run `31d25bfee9dd8db6b2887c2a6f3d7f41` completed on `4c1f7dde`
   with exit 101: formatting and Clippy passed, all 6376 library tests passed, and only
   two assertions failed across the whole workspace suite:
   `catalog::protocol_1_parses_with_schema_version_one` (89 versus 88) and
   `doctor_json_healthy_setup` (the newly added `notes` check). Both are owned by
   `bob-cli-ji`, and the catalog failure also has tracker `bob-cli-de`.

   No Swift edits are expected. Inspect completed macOS run `42928363199` for the
   integrated `df8045f` tree if it has settled; the prior phase run `42925346886` on
   `8028dbfe` passed the percent-tip and all double-tap tests but was red on known
   unrelated tests. The lander's source review confirmed intervening event editor
   changes, so do not claim those two whole trees are identical. If you discover a
   required Swift edit, compile it with `just portable-check`, follow the Mac scratch
   typecheck and exact-SHA CI rules, and use `/sase_monitor` for long commands. An
   unrelated failure stays with its recorded owner; any new epic-caused failure must be
   fixed before closeout.

4. **Finish the original epic's closeout now.** Re-read both phase statuses and their
   notes, the epic's audit/triage note, and any newer linked-plan descendants; confirm
   all remaining work above and both reviewer decisions are satisfied. Run
   `sase bead epic-symbols bob-cli-jt`. For every entry, wire it up, privatize it, add
   an appropriate non-test pragma, or delete it; re-key a whitelist line only when a
   still-open later bead genuinely needs it. There were no entries at the landing audit.
   Do not leave any entry keyed to this epic or a closing phase.

   Close `bob-cli-jt` normally with
   `sase bead close bob-cli-jt --note "<source/commit and drift verification, documentation corrections and counts, tests and limitations, and every follow-up disposition from the lander's triage note>"`.
   If the close reports open descendants, finish those first with their normal
   completion path; do not use `--force` to advance a successful nested landing. If it
   reports whitelist entries, resolve them and retry. Never force merely to make the
   command succeed.

   After closing, run `just symvision` if the recipe is available, otherwise record that
   it is unavailable (neither audited repo defines it). Open the plans repo with
   `/sase_repo`, read the original plan via `sase artifact read`, then set
   `status: done` in the frontmatter of
   `sase/repos/plans/202611/backslash_percent_alias.md`, using its freshly resolved
   path. Read `sase bead read bob-cli-jt -r "Need the parent link"` again: the confirmed
   parent is absent, so there is no ancestor closeout in this tale.

   This closeout must precede the turn's final declaration. Do not make it depend on
   this tale's own commit SHA, push, or CI result: close the epic in the same turn as
   the final documentation edits, then use `/sase_final` to commit every changed
   repository with the breaking marker on bob-cli's decision.
