---
tier: tale
size: medium
title:
  Cancel labels landing — list only real cancels, name the unblocked task, and close
  bob-cli-dl
goal:
  bob's capture cancel output matches the §17 contract (relation_cancels[] lists only
  real cancels, the capture row names a single unblocked task, the warning and ^prj
  strings follow CL6), and epic bob-cli-dl is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-dl.land
proposed_with: claude/opus
bead: bob-cli-dl
status: done
---

- **PARENT:**
  [202610/relation_cancel_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_cancel_labels.md)
- **BEAD:**
  [bob-cli-dl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dl/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-dl.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-dl.land.md)
- **COMMITS:**
  - [b8d5529](https://github.com/bobs-org/bob-cli/commit/b8d5529742330811a412d1364ea47eda1be8d5ef)
    — fix(capture): align cancel-label output with §17 CL6/CL7

# Cancel labels landing closeout (epic `bob-cli-dl`)

## Why this exists

The `bob-cli-dl` land agent checked all eight closed phases against the epic plan
(`plan:202610/relation_cancel_labels.md`, shown as PLAN by
`sase bead read bob-cli-dl -r "<why>"`) and the shipped code in bob-cli, bob-plugins,
and bob-mac-capture. The feature is in place and its suites are green. A few epic-caused
gaps remain, where bob's capture output disagrees with the normative §17 contract in
`docs/task-relations.md` (CL6 strings, CL7 JSON):

1. **`relation_cancels[]` also lists sources that were not cancelled.** When the cancel
   label lands on an already-closed source (Done or Cancelled), bob still emits a
   `relation_cancels[]` row whose message is `⊘ Stays Done · link only` or
   `⊘ Already Cancelled · link only` (`src/native/capture/relations_cancel.rs`,
   `already_closed_message`, and the second `Ok(Some(RelationCancelJson{..}))` in
   `plan_source_cancel`).
   - Bob Mac Capture counts every `relation_cancels[]` row for its
     `Capture & cancel N task(s)` submit hint
     (`CaptureRelationCancelPresentation.cancelCount`). It also renders each row in the
     ⊘ cancelled style.
   - So relinking an already-cancelled task with `duplicate of` tells Bryan the capture
     will cancel a task when it will not.
   - The plan intended already-closed sources to get "the link only, with
     `relation_updates[].cancel_effect: "already_closed"`". The CL6 already-closed text
     is muted, with no ⊘: `Already Cancelled · link only` / `Stays Done · link only`.
     This is how capture-complete and nav already word it.
2. **The capture row does not name the unblocked task.** CL6 says `… · unblocked “{D}”`,
   and the epic's headline example is
   `⊘ would cancel “Write introduction” · Next → Cancelled · would remove 1 Pomodoro link · would unblock “Finish full v1 draft”`.
   - `cancel_message` prints `unblocked {count}` / `would unblock {count}`.
   - The Mac preview renders bob's message verbatim, so it shows `would unblock 1`.
3. **The structured warning quotes the label.** The code emits
   `⊘ cancelled “{S}” (“{label}” “{T}”)`, but CL6 says
   `⊘ cancelled “{S}” ({label} “{T}”)`. Same for the dry-run `⊘ would cancel …`.
4. **The `^prj` suffix ignores the singular.** CL6 says "Plurals follow the count".
   - Capture prints `its 1 open tasks stay open`.
   - capture-complete (`src/native/capture_complete/cancel.rs`) prints
     `its 1 open task stay open`.
   - The nav consequence strip (bob-plugins
     `plugins/bob-navigation-hotkeys/src/305-relation-stage-copy.js`, around line 175)
     prints `its 1 open tasks stay open`.

Everything else was verified and needs no change. That includes the LG16 `~` flip, which
landed during the epic; both clients re-derive the source from the current direction, so
it needs no integration work. Do **not** touch bob-mac-capture. The Mac already renders
whatever bob says. Its macOS CI is red for an unrelated reason (task `bob-cli-dr`), and
its decode fixtures `relation-cancel-dry-run.json` and `relation-cancel-commit.json`
stay valid inputs, because the Mac skips that warning whenever `relation_cancels[]` is
non-empty.

Rules that still apply (from the epic plan):

- Never edit `Cargo.toml` versions or `CHANGELOG.md`.
- Use `fix(capture): …` / `fix(nav): …` style subjects, with no `!`.
- The JSON stays additive, and nothing changes when `relations.cancel_labels` is absent.
- Before touching JSON contracts or docs, read `cli_rules.md` and `docs.md` with
  `/sase_memory_read`.
- Open bob-plugins only through `/sase_repo` and read its `AGENTS.md`.
- `~/bob` is read-only; scratch vaults use `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, and
  `BOB_CONFIG_FILE`.

## Steps

### 1. bob-cli: `relation_cancels[]` lists only sources the item cancels

- Keep the already-closed outcome internal, used only for the human line. One option:
  - Add a `#[serde(skip)]` `effect: RelationCancelEffect` field to `RelationCancelJson`
    (`src/native/capture/output.rs`).
  - Serialize `relation_cancels` through a filter that keeps only `Cancelled` rows.
  - Skip the field when no `Cancelled` row remains.

  Any equivalent that adds no new JSON field works too.

- Already-closed sources still mark `relation_updates[].cancel_effect: "already_closed"`
  (unchanged), and still emit no `relation_cancelled_source` warning.
  - Replace the string-prefix test `is_link_only_message` with the effect.
- Human output (`print_human_relation_updates`) prints the CL6 muted line, with no ⊘:
  - `Already Cancelled · link only` for `-`;
  - `Stays Done · link only` for every other closed symbol (match capture-complete's
    rule).
  - Drop the `⊘ Stays {status_name}` fallback.
- Update the `RelationCancelJson` / `relation_cancels` doc comments ("one row per source
  this item cancelled").

### 2. bob-cli: CL6 strings in `cancel_message` and the warning

- **Recovered clause:**
  - exactly one recovered dependent → ` · unblocked “{D}”` (dry run
    ` · would unblock “{D}”`), where `{D}` is that row's clean `text` (the same text
    `recovered[]` carries);
  - two or more → ` · unblocked {k} tasks` / ` · would unblock {k} tasks`.
- **`^prj` suffix:**
  - N == 1 → ` · its 1 open task stays open`;
  - otherwise → ` · its {N} open tasks stay open`.
  - Apply this in `cancel_message` and in capture-complete's `cancel.rs`.
- **Structured warning:** `⊘ cancelled “{S}” ({label} “{T}”)` and
  `⊘ would cancel “{S}” ({label} “{T}”)` (label unquoted).

### 3. Tests (bob-cli)

- `tests/cli/capture/relations_cancel.rs`:
  - Update the asserted strings near lines 254, 261, and 817. The 817 resident recovery
    now names the resident.
  - `cancel_already_closed_links_only` asserts that `relation_cancels` is absent from
    the JSON, that `cancel_effect` is still `already_closed`, and that the human output
    carries the muted line without ⊘.
- Add CLI cases for:
  - two recovered dependents → `unblocked 2 tasks`;
  - a dry run naming one dependent (`would unblock “…”`);
  - a `^prj` source with exactly one open resident → `its 1 open task stays open`.
- `src/native/capture_complete/tests/relations.rs`: add the singular `^prj` case next to
  the existing `its 2 open tasks stay open` assertion (line 629).
- Keep the inert-without-config test byte-identical.

### 4. Docs (bob-cli)

- `docs/task-relations.md` §17:
  - CL6: the capture row text spells out the one-vs-many `unblocked` forms. The `^prj`
    suffix row shows the singular, and the already-closed row notes that capture prints
    it as a muted human line only.
  - CL7: `relation_cancels[]` holds one row per source the item cancels; already-closed
    sources appear only as `relation_updates[].cancel_effect`.
- `docs/capture.md` (around lines 2395–2402): update the example row to name the
  dependent, and the already-closed sentence.
- `docs/reference/json-outputs.md`: the `bob capture` row says `relation_cancels[]`
  lists cancelled sources.
- Run `just docs-gen` if any help text changes (none expected), then `just docs-check`.

### 5. bob-plugins: nav `^prj` singular

- Open with `/sase_repo` (`bob-plugins`).
- In `plugins/bob-navigation-hotkeys/src/305-relation-stage-copy.js`, use
  `its 1 open task stays open` when `projectOpenTasks === 1`.
- Add or extend the copy test in `scripts/test-navigation-relation-links-stage.cjs`.
- Run `npm run build`, `npm test`, and `bob plugins sync`. If sync refuses because of
  another checkout's uncommitted deploy, do not force it; record that in the close note.
- Commit the plugins change.

### 6. Verify

- `cargo test --test cli -- relations_cancel relations:: relations_rename capture::parse_relation`
- `cargo test --lib -- task_cancel capture_complete`
- `just check` (not `just check-full`)
- `just docs-check`
- In a scratch vault with the CL2 config, `bob capture --dry-run -f json` on an
  already-cancelled source prints no `relation_cancels`, and `cancel_effect` is
  `already_closed`.

### 7. Close out epic `bob-cli-dl` (final step, same turn as the code)

1. Run `sase bead epic-symbols bob-cli-dl`. It listed no entries at landing time. Wire
   up, privatize, or delete any symbol that appears, per the Symvision epic-whitelist
   policy. Re-key a Justfile line only to a still-open bead that needs it.
2. Close the epic with: `sase bead close bob-cli-dl --note "<verification>"`. Never use
   `--force` merely to make the close succeed. The note must summarize:
   - the land agent's verification: all 8 phases checked against the code; vectors
     byte-identical in bob-cli and bob-plugins; the chezmoi `relations.cancel_labels`
     commit b33aa49d is on origin; macOS CI run 40729693701 built the cancel cue and
     passed `CaptureRelationCancelTests` (its only failures belong to `bob-cli-dr`);
     LG16 needed no integration;
   - this tale's fixes and the commands that passed;
   - the follow-up outcomes already recorded on `bob-cli-dl`: tasks `bob-cli-dq`,
     `bob-cli-dr`, `bob-cli-ds`; a DISCOVERED ISSUE note on `bob-cli-bj`; the
     cancel-stage door declined per `cancel_stage_door = no`; the rest already fixed
     upstream.
3. Run `just symvision` and confirm the whitelist is clean.
4. In the epic's plan file (the PLAN path printed by `sase bead read bob-cli-dl`,
   `plans/202610/relation_cancel_labels.md` in the plans sidecar), change frontmatter
   `status: wip` to `status: done`.
5. `bob-cli-dl` has no `parent_bead`, so nothing further cascades.
