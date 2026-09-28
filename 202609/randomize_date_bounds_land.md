---
tier: tale
size: small
title: Guard randomize date bounds and land bob-cli-2b
goal:
  "`bob randomize` rejects unrepresentable `--until` and priority-window dates without
  wrapping or panicking, then bob-cli-2b is closed after verification."
bead: bob-cli-2b
proposed_by: bbugyi200.apollo.bob-cli-2b.land
status: done
---

- **PARENT:**
  [202609/bob_randomize.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/bob_randomize.md)
- **BEAD:**
  [bob-cli-2b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2b/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2b.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2b.land.md)
- **COMMITS:**
  - [8487fe2](https://github.com/bobs-org/bob-cli/commit/8487fe28dba27ad396b135a87c172b4b54a41a39)
    — fix(randomize): reject unrepresentable until offsets and priority rolls without
    wrapping

# Guard randomize dates and land bob-cli-2b

`bob-cli-2b` implemented `bob randomize` in four closed phases. Its land audit found
that `src/native/randomize.rs::parse_until` casts an arbitrary `u64` `+N` to `i64` and
adds it to `NaiveDate`. For example, `+18446744073709551615` becomes negative one,
violating the CLI contract that `--until` is today or later. The pure planner also adds
the configured `u64` roll offset to `until` without a checked date range; an extreme
priority window or cutoff can panic. A direct run with that `+18446744073709551615`
value reached config loading (exit 1 for a deliberately missing config) instead of
returning usage exit 2, confirming the invalid value was accepted. This is remaining
epic work. The only child `PROPOSED FOLLOW-UP` is the same pre-existing Clippy `|| true`
deny in `tests/cli.rs`, introduced by `22abed4`; the lander recorded all four proposals
on `bob-cli-2b` and corroborated the owner epic `bob-cli-28`. No additional task needs
filing.

1. Make `--until +N` reject any offset that does not fit in the supported date range,
   returning the existing usage-error exit code 2. Keep ordinary `+N` and ISO dates
   unchanged. Replace unchecked planner date addition with a checked path so a cutoff
   plus any configured priority roll either produces a valid date or reports a
   deterministic error before any write. Include the 35-day load horizon in the boundary
   handling. Preserve the existing JSON failure contract for runtime/config errors and
   avoid partial writes.
2. Add focused tests for the oversized `+N`, a date at the representable boundary, and
   an extreme configured priority window. Verify normal seeded rolls and JSON output
   remain stable. Run the relevant cargo tests and formatting. Run `just check` as the
   requested file-change verification; this checkout currently reports that the recipe
   does not exist, so report that infrastructure limitation and use the available
   focused test/fmt recipes. Do not run `just check-full`.
3. Recheck source and the interleaved capture/Pomodoro changes against the approved
   `plan:202609/bob_randomize.md`. Four epic commits are `b4b51ea`, `f17339d`,
   `1e8484b`, and `35c6ba4`; interleaved `fe2c0b8`, `0dfbc55`, and `b10b45e` shift
   Pomodoro times or document them without changing the open daily-note task-link
   structure used by randomize. Check for any newer drift before closing. The
   `bob-cli-2b` child notes and follow-up triage are already recorded; address any
   further epic-caused issues before close.
4. Finish this epic's closeout: run `sase bead epic-symbols bob-cli-2b`, resolve or
   re-key every listed entry to a still-open later bead, then close normally with
   `sase bead close bob-cli-2b --note "<verification of phases, integration, tests, and follow-up triage>"`.
   Do not use `--force` merely to get a close. Run `just symvision` if available to
   confirm a clean whitelist. Set `status: done` in the frontmatter of
   `plan:202609/bob_randomize.md` (open the `plans` sidecar via `/sase_repo` before
   editing). Re-read `bob-cli-2b` for `parent_bead`; the current bead has no parent, so
   no ancestor close is expected. Declare changed repositories through `/sase_final`.
