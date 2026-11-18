---
tier: tale
title: Show the lane-excess count next to the Mac Capture w mark
goal:
  Mac Capture's lit w mark shows Bob's count of lane tasks over their caps, the fewest
  task changes that silence w.
size: medium
decisions:
  memory_w_count_supersession:
    ask:
      Should the decisions web record the w count as a new strand that partly supersedes
      the bare-w claim?
    memory:
      - decisions:idle-capture-mirrors-walk-attention
      - decisions:w-count-reads-lane-excess
    default: true
    corrects: "`w` stays bare because its report label names the over-limit lanes."
    answer: true
proposed_by: bbugyi200.apollo.f1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 13:27:33
status: wip
---

# Show the lane-excess count next to the Mac Capture `w` mark

## Goal

Bob Mac Capture's idle agenda title row shows walk-attention marks. Today `s` and `b`
carry a count, and `w` is shown bare. Make `w` show a count: the number of tasks that
must change before the `w` walk goes quiet.

Example: caps WIP 10 and NEXT 20, with 11 WIP and 22 NEXT tasks, show `w 3`.

### Definition

`excess = Σ max(0, count − cap)` over the three `w` lanes, in lane order: `pending`
(WIP), `next`, and `ready`.

- Lanes under their cap add 0. They never subtract.
- READY counts too. A READY lane over its cap lights `w` by itself, and `]w` walks WIP,
  NEXT, and READY.
- This sum is the smallest number of task changes that silences `w`:
  - Each lane task is in exactly one lane.
  - Each change removes at most one task from one lane. Examples of a change: close the
    task, link it to Today, or move it down a rung. A demotion only helps when the lower
    lane has room.
  - So silencing `w` takes at least the sum, and closing or Today-linking that many
    tasks is always enough.
- `attention` is true exactly when `excess > 0`. A lit `w` therefore always shows at
  least `1`, and a calm `w` reports `0`.

## Contract (bob-cli owns it)

- An **available** `w` family in `bob walks --format json` gains
  `"counts": {"excess": N}` next to its existing `lanes` array.
  - This is the same shape `s` uses (`counts.walk`) and `b` uses (`counts.waiting`).
  - The field is additive and optional, so `schema_version` stays `1`. `docs/walks.md`
    says: "New optional fields keep version 1".
  - An unavailable `w` omits `counts`, like every other family extra.
- These stay byte-for-byte unchanged: `attention`, `tier`, `label`, the human output,
  and the `bob walks --help` text. That also leaves the generated
  `docs/reference/cli/walks.md` unchanged.
  - The label must still equal the walker footer's push label.
  - bob-plugins needs no change. Its walk-attention vectors are hand-copied in
    `scripts/test-ledger-tools-walk-attention-vectors.cjs`, and it never reads this
    JSON.
- Mac Capture reads `counts.excess` for `w` and nothing else. Swift must not compute the
  count from `lanes`, because Capture is a thin client and Bob owns the attention facts
  (see `decisions:mac-capture-is-a-thin-client` and claim 1 of
  `decisions:idle-capture-mirrors-walk-attention`).
  - An older bob without the field yields `count == nil`, which keeps today's bare `w`.

Land bob-cli first (Part 1), then Mac Capture (Part 2), then memory (Part 3).

## Part 1 — bob-cli

### `src/native/walks/mod.rs`

1. Change `FamilyBody::Lanes { lanes }` to `FamilyBody::Lanes { lanes, excess: u32 }`.
2. Add `fn lanes_excess(lanes: &[LaneCount]) -> u32`. It sums
   `lane.count.saturating_sub(lane.cap)`.
3. In `lanes_family`, compute `lanes_excess(&lanes)` and store it in the body.
4. In `family_json`, for the `Lanes` arm, insert `"counts": {"excess": excess}` next to
   `"lanes"`.
5. Extend the module doc comment with one line saying that `w` reports `counts.excess`,
   which is the Mac `w` count.
6. Add a unit test in the existing `tests` module:
   - lanes `12/10`, `3/15`, `101/100` give `3`;
   - `10/10`, `0/15`, `0/100` give `0`;
   - over-lane presence agrees with `excess > 0` in both cases.

### `tests/cli/walks.rs`

1. **W1** (`w1_pending_over_cap_is_on`): also assert `w["counts"]["excess"] == 1`.
2. **W2**: assert excess `0`.
3. **W3**: assert excess `0`.
4. **W8**: assert excess `1`.
5. Add **W9**, `w9_excess_sums_every_over_lane`. It mirrors the user's example exactly:
   - Write a config file `plan:\n  max_next: 20\n` and pass it through
     `BOB_CONFIG_FILE`, as `invalid_plan_config_makes_w_unavailable` does.
   - Write 11 pending rows: `n_lines("Pending", "/", 11, " [fresh:: 2026-11-17]")`.
   - Write 22 next rows: `n_lines("Next", "*", 22, "")`.
   - Expect:
     - `attention == true`;
     - `label == "Lanes over limit: WIP 11/10 · NEXT 22/20"`;
     - `lanes[1]["cap"] == 20`;
     - `counts.excess == 3`.
6. Regenerate the goldens in `tests/fixtures/walks/` (`walks-clear.json`,
   `walks-all.json`, `walks-bank-unavailable.json`). Run
   `BOB_UPDATE_GOLDENS=1 cargo test --test cli walks`, then rerun without the variable.
   - Check the diff. The only change allowed is a `w` `"counts": {"excess": N}` block:
     `0` in clear, `1` in all (NEXT 16/15), and whatever bank-unavailable's lanes imply.

### `docs/walks.md` (the shared contract)

1. In **w — lanes**, add an **excess** bullet:
   - `Σ max(0, count − cap)` over all three lanes;
   - the fewest task changes that silence `w`;
   - `attention` iff `excess > 0`.
   - End the section with `counts.excess is the Mac w count.` (format `counts.excess`
     and `w` as code). This mirrors the `s` section's "`counts.walk` is the Mac `s`
     count."
2. In **JSON (schema_version 1)**, change the sentence to: an available `w` carries
   `"counts": {"excess": N}` and `lanes`. Add `"counts": { "excess": 2 }` to the
   example's `w` entry (WIP 12/10).
3. Update **Conformance vectors**:
   - W1 expects `excess 1`.
   - W2 expects `excess 0`.
   - W8 expects `excess 1`.
   - Add a W9 row:
     `PENDING 11, NEXT 22, plan.max_next 20 | on, WIP 11/10 · NEXT 22/20, excess 3`.
   - Add one sentence: `excess` expectations are bob-only, because the walker footer
     renders no `w` count.

### Rules and gates

- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`. Releases ride the daily train.
  Use a conventional `feat(walks): …` message. This is not a breaking change.
- Run `just check`: fmt, clippy gate, and every test binary.
- Run `just docs-check`.

## Part 2 — bob-mac-capture

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture -r "…"`). Read its
`AGENTS.md` before editing, and use only the path the command prints.

### Sources

1. `Sources/CaptureCore/WalkAttentionModels.swift`
   - Add `let excess: Int?` to `WalkAttentionCounts`.
   - In `WalkAttention.init(from:)`, change the `.lanes` case to
     `count = counts.excess`. `.other` stays `nil`.
   - Update three doc comments that say `w` stays bare: the struct comment, the `count`
     property comment, and the decode comment. Each should say that `w` carries Bob's
     lane excess (`counts.excess`), and nil comes from an older bob.
2. `Sources/CaptureCore/CaptureAgendaPresentation.swift`: update the
   `CaptureAgendaWalkMark` doc comment the same way. Keep the "Bare frame (no `[[`
   brackets)" sentence, which is about brackets, not counts.
3. `Sources/BobMacCapture/CaptureAgendaView.swift`: in the `WalkMarksView` doc comment,
   change `s`/`b` count to `s`/`w`/`b` count. This is a comment-only change.
   - `markCell` already renders `mark.count` for any family, so no view logic changes.
   - The fit planner's measure key already includes the marks, so no change there
     either.

### Fixtures

Replace `Tests/Fixtures/walks-all.json`, `walks-clear.json`, and
`walks-bank-unavailable.json` with Part 1's regenerated bob-cli goldens, byte for byte.
They are identical copies today. Leave `fake-bob` unchanged.

### Tests

1. `Tests/CaptureCoreTests/WalkAttentionDecodingTests.swift`
   - `testAllFixtureDecodesThreeFamiliesInOrder`: expect `walks[1].count == 1` instead
     of `nil`.
   - Add a test for a `w` entry with no `counts` (older bob). It decodes `count == nil`.
   - Add a test for a `w` entry whose `counts` has only `walk`. It also decodes `nil`,
     because keys are per family.
2. `Tests/CaptureCoreTests/WalkAttentionPresentationTests.swift`,
   `testMarksAreInSelectorOrderWithCounts`: expect `marks[1].count == 1` and fix the "w
   stays a bare mark" comment.
3. `Tests/BobMacCaptureTests/CaptureAgendaDesignTests.swift` (the `.lanes` entry near
   line 282) and `Tests/BobMacCaptureTests/CaptureAgendaHeightConsistencyTests.swift`
   (near line 156): pass `count: 1` to the lanes `WalkAttention`. The synthetic renders
   and the height checks then cover the `w 1` look. The row height must not change.

### Docs (mounted into Bob's site)

1. `docs/idle-agenda.md`, **Walk attention**: replace "and `w` stays bare because its
   label already names the over-limit lanes" with this text:
   - `w` shows how many lane tasks sit over their caps: Bob's `counts.excess`, the
     fewest task changes that silence `w`.
   - Example: WIP 11/10 and NEXT 22/20 show `w 3`.
   - Older Bob versions without that count show a bare `w`.
2. `docs/protocol.md`: in the `walks` `counts` bullet (around line 300), add
   `w=`excess``.

### Gates (from that repo's AGENTS.md)

1. Run `just portable-check` on a host that has Swift. This host may have none; athena
   does.
2. `CaptureAgendaView.swift` and `Tests/BobMacCaptureTests` changed, so do a best-effort
   scratch compile on the mac:
   - rsync the repo without `.build` and `.git`;
   - run `bundle.sh`, then `just typecheck-tests`;
   - delete the copy.
   - Never use Bryan's checkout or `just install`.
3. Run `just docs-check`.
4. Run `Scripts/macos-ci-check` before landing.
5. After the push, use `/sase_monitor` to watch the exact-SHA macOS run until it is
   green.
6. If the mac is offline, say in the final report that the macOS targets are unverified.
7. Optional: if the green master run publishes synthetic renders, `just docs-media` can
   import refreshed idle-agenda images, as `c44dd878` did. Do not block on this.

## Part 3 — memory (decisions web)

Use `/sase_memory_write` before touching memory, then run `sase memory init`. Never
hand-edit `AGENTS.md`.

> [!decision] memory_w_count_supersession Add a new `decisions` web strand with slug
> `w-count-reads-lane-excess` (selector `decisions:w-count-reads-lane-excess`).
>
> - Frontmatter follows `decisions/review-attention-follows-the-walk.md`: `keyword`
>   (e.g. "The Mac `w` Mark Counts Lane Excess"), a short `summary`, and
>   `metadata: {status: accepted, decided: <date>}`.
> - **Applies to:** bob-cli `bob walks` and `docs/walks.md`, and Bob Mac Capture's
>   idle-agenda `w` mark.
> - **Claim:** an available `w` reports `counts.excess = Σ max(0, count − cap)` over
>   WIP/NEXT/READY, which is the fewest task changes that silence `w`. The Mac `w` shows
>   it the way `s` shows `counts.walk`. An older bob without it keeps a bare `w`. The
>   label and walker footer are unchanged.
> - **Why:** Bryan's 2026-11-18 request, including the 11/10 + 22/20 → `3` example.
>   Rejected alternatives:
>   - keeping `w` bare;
>   - Swift summing `lanes` (it breaks the thin-client rule);
>   - counting only WIP and NEXT (READY over its cap also lights `w`);
>   - putting the number in the shared label (it breaks byte-equality with the walker
>     footer).
> - **Cost:** one more JSON field.
> - **Evidence:** the approved plan ref, `docs/walks.md` W9, and commit SHAs and the
>   green exact-SHA macOS CI run, as available.
> - **Reopens when:** the `w` predicate changes, a lane joins or leaves `w`, or the
>   footer gains its own `w` count.
>
> Link back to `[[decisions:idle-capture-mirrors-walk-attention]]`.

> [!decision] memory_w_count_supersession In
> `sase/memory/decisions/idle-capture-mirrors-walk-attention.md`:
>
> - Keep `metadata.status: superseded-in-part`.
> - Append `decisions:w-count-reads-lane-excess` to `superseded_by`.
> - Append one back-link paragraph after the existing "Superseded in part" paragraph. It
>   should say that `[[decisions:w-count-reads-lane-excess]]` retires the bare-`w` half
>   of claim 3, and that `w` now shows Bob's `counts.excess`.
> - Do not reword the accepted body.

If the memory decision was turned off and no human reviewed the plan, file a `memory`
task bead through `/sase_new_task` instead, as `/sase_memory_write` describes.

## Acceptance

- On today's vault, `bob walks -f json` shows `w` `counts.excess` equal to the sum of
  `count − cap` over the over-cap lanes. The human output is byte-identical to before.
- `just check` and `just docs-check` pass in bob-cli. The W9 vector reproduces the
  user's example (`3`).
- In Mac Capture, a lit `w` shows Bob's excess beside the letter, the same way `s 4`
  shows its count. An older bob still shows a bare `w`. Portable checks pass, and the
  exact-SHA macOS CI run is green.
- The decisions web records the new rule, and the old record points to it.
