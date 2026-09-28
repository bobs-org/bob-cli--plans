---
tier: tale
title: Correct the shift CLI example and close bob-cli-2a
goal: The "Shifting the current Pomodoro" example in docs/capture.md shows a command
  that actually shifts one unit earlier, and epic bob-cli-2a is closed with its plan
  marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-2a.land
bead: bob-cli-2a
status: done
---

- **PARENT:**
  [202609/pomodoro_shift_operators.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/pomodoro_shift_operators.md)
- **BEAD:**
  [bob-cli-2a](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2a/README.md)

# Correct the shift CLI example and close bob-cli-2a

The Pomodoro shift feature is already implemented. This tale only corrects one docs
example that the land audit found, then closes epic bob-cli-2a. Do not reopen the
feature, do not touch bob-mac-capture or bob-plugins, and do not change how `bob` treats
`--`.

## Fix the example

In `docs/capture.md`, section "Shifting the current Pomodoro", the fenced example is:

```bash
bob capture ++3
bob capture -- --2
bob capture --
printf -- '--2\n\n+\n' | bob capture
```

`bob capture --` is the end-of-options marker and submits no text, so it does not shift.
The next paragraph already says that, and it documents `bob capture --1` as the one-unit
earlier shift. `bob capture --help` in `src/native/capture.rs` already teaches `--1` and
`bob capture -- --2`, and `capture_parse_pomodoro_shift_human_and_help` asserts those
strings.

Replace only the bare `bob capture --` line in that fence with `bob capture --1`. Leave
the following paragraph, including its explanation that a bare `--` carries no text,
unchanged. Do not add or remove other examples.

## Do not do this

- Do not "fix" the pre-existing `|| true` clippy deny at `tests/cli.rs:31815`. Phases
  bob-cli-2a.1 and bob-cli-2a.2 proposed it. It is the solo-link assertion from
  in-progress epic bob-cli-28 (commit 22abed4), already recorded there as a discovered
  issue. The land agent declined a new task and noted bob-cli-2a. Leave that assertion
  alone.
- Do not integrate the randomize commits `b4b51ea` and `f17339d`. They do not touch the
  capture-operator files and do not duplicate shift math.
- Do not run `just check-full`. There is no `just check` recipe in this justfile. This
  tale changes one Markdown example; do not run the full Rust suite for that.

## Close the epic

The land audit already verified the feature. Use that result in the close note. Before
closing:

1. Run `sase bead epic-symbols bob-cli-2a`. The land audit found no entries. If any
   `--epic-symbol` entry is listed, resolve it (wire it up, privatize it, add a non-test
   pragma, or delete it per the Symvision epic-whitelist policy) or, only when a
   still-open later bead still needs the exemption, re-key that Justfile line to that
   open bead. `sase bead close` refuses while any entry keyed to this epic remains. Do
   not leave that judgment.

2. Close with this note (one line added for the docs edit is enough if the example
   change landed):

```text
Verified bob-cli-2a against phases 2a.1, 2a.2, and 2a.3, plan plan:202609/pomodoro_shift_operators.md, and the source.

CLI commits fe2c0b8 (bob-cli-2a.1) and 0dfbc55 (bob-cli-2a.2): session-operator lexer (one sign resizes, two signs shift, omitted count is 1), PomodoroShift kind, staged planner translates both endpoints with Euclidean mod 1440 and keeps duration, matching bob-plugins offsetPomodoroLineRange. JSON pomodoro_shift, human shifted/would-shift output, capture-parse mode/span/spec and invalid_pomodoro_shift, @@/completion/rewrite skip operators, and help text are in place. CLI tests cover JSON, bare defaults, both midnight wraps, metadata/CRLF/children, stopwatch and range-span fallbacks, the canonical ++3 line, ordered batches, dry-run rollback, errors, prose non-matches, and the --3/-/--1 argv spellings. The land agent re-ran `cargo test --test cli shift` on 0dfbc55: 10 passed, 0 failed.

Mac commit 794f06a on bob-mac-capture (bob-cli-2a.3): tolerant shift decode, double-chevron presentation, Shift and Adjust footer titles, notification and palette mapping, smart dashes disabled with the Tab em-dash note, and real-bob fixtures. GitHub Actions run 36444782814 succeeded on that SHA.

Integration: after fe2c0b8 the only non-epic bob-cli commits are b4b51ea and f17339d (bob-cli-2b randomize). They do not touch capture operator files. No bob-mac-capture commit other than 794f06a landed after the epic started. No integration edit was required.

Docs: replaced the shift example's bare `bob capture --` with `bob capture --1`, because a bare `--` submits no text.

Follow-ups: bob-cli-2a.1 and bob-cli-2a.2 proposed the pre-existing tests/cli.rs `|| true` clippy deny (line 31815 on 0dfbc55). Not caused by this epic. No task bead matches it. bob-cli-v covers warnings only. It belongs to in-progress bob-cli-28 (22abed4); corroborated with a DISCOVERED ISSUE on that epic. bob-cli-2a.3 proposed nothing. No parent bead. epic-symbols: none.
```

Command:

```bash
sase bead close bob-cli-2a --note "<the note above>"
```

Never use `--force` merely to make the close succeed. If the close is rejected because
`--epic-symbol` entries remain, clean those up and close again. If it is rejected
because named phases were never completed, finish or reopen them, or record the outcome
with `--force --reason ... --resolution canceled|superseded` only when that outcome is
true. Do not force a successful nested landing.

3. Run `just symvision` if that recipe exists. This checkout's justfile has no
   `symvision` recipe. If `just symvision` reports that the recipe is missing, record
   that in a short follow-up note on bob-cli-2a if the close note did not already say
   the recipe is absent. Do not add a recipe. If the recipe exists and fails because of
   a whitelist entry for this epic, fix that entry and rerun until it is clean.

4. In the epic plan file `sase/repos/plans/202609/pomodoro_shift_operators.md` (the PLAN
   path from `sase bead read bob-cli-2a`), set the frontmatter field `status: done`.
   Change only that field.

5. Re-read the closed epic with
   `sase bead read bob-cli-2a -r "Confirm close and parent link"`. The land audit found
   no parent bead. If that read still shows no parent, stop. If it shows a parent,
   handle that parent using the land-agent parent rules and the concrete parent ID from
   the read: a phase parent is closed only when this child completed that phase's work;
   a plan parent is rechecked and closed only while it remains fully complete, after its
   own epic-symbol cleanup, `just symvision`, and plan-file `status: done`. Stop at the
   first incomplete or ambiguous parent and record the blocker on that parent.
