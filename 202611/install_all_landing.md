---
tier: tale
size: medium
title:
  "Land bob-cli-i0: real Cargo phases, aligned rows, safe fallbacks, and the install
  guide"
goal:
  A real `just install` and `just install-all` show the right Cargo phase at each moment
  and record build time under the phase it belongs to. Every runner and digest row sits
  on the plan's 2-space indent. A display or helper failure can never re-run or fail a
  build that succeeded. The install guide documents the new run. Epic bob-cli-i0 is
  closed and its plan is marked done.
proposed_by: bbugyi200.athena.bob-cli-i0.land
proposed_with: claude/opus
bead: bob-cli-i0
create_time: 2026-11-06 18:14:57
status: wip
---

- **PARENT:**
  [202611/install_all_wait_and_whats_new.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/install_all_wait_and_whats_new.md)
- **BEAD:**
  [bob-cli-i0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-i0/README.md)

# Plan: land epic bob-cli-i0 (`just install-all`, a wait worth sitting through)

## Why

The landing review of epic `bob-cli-i0` found that its phases are closed but a few of
its contracts are not met yet. Its plan is
`plan:202611/install_all_wait_and_whats_new.md`. Read that plan's "The live runner",
"The bash run shim", Screens A–F, and "Phase `docs-acceptance`" before starting. This
tale finishes that work and then closes the epic. It is the epic's landing, so its last
step closes `bob-cli-i0`.

What the review found:

1. **Real Cargo builds report the wrong phase for the whole build.** `cargo install`
   prints `Installing bob-obsidian v0.4.0 (/path/to/bob-cli)` as its _first_ line.
   `parsers.py` maps every `Installing` line to the `finish` phase, and the runner's
   `_PhaseClock.advance` only moves forward. So the live line reads `installing bob`
   from 0:00 to the end, and `runs.jsonl` puts all of the build's seconds under
   `finish`, with `index`, `deps`, and `crate` near 0. Bob-cli-i0.7's real-Cargo
   acceptance shows this: its runner log starts with that `Installing` line, and its
   history rows record `index 0.08 s`, `deps 0`, `crate 0`, `finish 1.0 s`. The same
   transcript shows that `       Fetch [====>   ] 246 complete; 3 pending` progress
   frames and the
   `   Replaced package`…`with`…``line leak into scrollback. Only`Building
   [`frames and`Replacing`are chatter today. No test fixture contains the opening`Installing
   <crate> v<ver> (<path>)` line.
2. **Rows sit at column 0, but the plan puts them under a 2-space indent.** Screens A–D,
   and every other install-all row (`  ⇣  bob-cli`, `  $ just …`,
   `  ✓ 3 new commits …`), use a 2-space indent. The runner's live frame, its
   `✓ built · …`, `✗ build failed after …`, and `✗ interrupted after …` rows, the
   digest's repo header and commit rows, and the summary replay rows all print flush
   left.
3. **Two fallback gaps break "a display failure never changes an install outcome."**
   - The `install_ui_run` shim in `scripts/lib/install_ui.sh` re-runs the command
     whenever the started file is missing, even when the runner exited 0. The plan
     allows the fallback only when the runner exited nonzero _and_ the started file was
     never created.
   - After the child is spawned, an unexpected exception in `live.run` changes the
     result. That covers the parser, the history write, the estimate, and the final
     rows. The exception reaches `__main__.main`, which returns 70, so a successful
     build reports failure. If the exception hits mid-stream, `proc.wait()` runs while
     nothing drains the pipe, and a chatty child can block forever.
4. **The bob-cli summary duration grew.** `cli_start=0` in `scripts/install_all`. Now
   that the sibling pulls, digest, and plan line run before the bob-cli build, its
   summary duration includes all of them. Screen C shows the bob-cli summary duration
   equal to its `built` time.
5. **Plain `just install` gets the wrong failure hint.** The runner's failure rows
   always say `rerun with  just install-all -v`, even when the user ran `just install`.
6. **docs-acceptance is incomplete.**
   - `docs/guides/install-everything.md` still shows the stale `installed`/`synced`
     sample rows that the plan said to remove.
   - The guide has no sample run, no explanation of the ETA phrases, no escape-hatch
     variables, and no log or state locations.
   - The sandboxed `scripts/install_all` acceptance transcript was never captured.

Integration check: the only non-epic commits since the epic started (`ae624e27`,
`f1d8285e`, `e39e4576`, and upstream `17aae714`) touch gmail and capture code only.
Nothing outside the files below needs to change. The landing agent already filed the
follow-ups: `bob-cli-i6` (Mac acceptance), `bob-cli-i7` (Mac build speed), and
`bob-cli-i8` (ETA gate tuning, snoozed a week). It recorded the outcomes on
`bob-cli-i0`. Do not file them again.

## Constraints (unchanged from the epic)

- Python is stdlib only, Python 3.9 syntax, and every module starts with
  `from __future__ import annotations`. Bash stays 3.2-safe. `test_portability.py`
  enforces both.
- Commit subjects use `fix(install):`, `test(install):`, or `docs(install):`. Do not
  edit the `Cargo.toml` version or `CHANGELOG.md`.
- This is not a CLI or JSON contract break, so no `!`. History rows keep `v: 1`.
- Do not migrate existing `runs.jsonl` rows. Rows recorded with the wrong phase split
  leave the newest-10 sample window after about ten corrected builds. Until then the
  range rung simply withholds an estimate, and the `last similar build` rung, which uses
  step totals, stays correct.

## Steps

### 1. Cargo phases that match real `cargo install` output (`parsers.py`, `live.py`)

- Cargo's opening line, `Installing <name> v<version> (<source>)`, is chatter with no
  phase change: the build stays in `index`. Only the later `Installing <absolute path>`
  lines map to `finish`, along with `Replacing`, `Replaced`, and `Installed`. `Finished`
  keeps mapping to `finish`.
- Every Cargo progress frame of the form `<Word> [` (`Building [`, `Fetch [`,
  `Downloading [`, and so on, after leading spaces) is chatter.
  - `Building [` keeps its current deps/crate and `n/m` counter logic.
  - Other frames map to `index`, or to no phase. They must never advance past the
    current phase.
- Keep the patterns in the single `CHATTER_PATTERNS` table.
- The `index`-phase label must never echo raw line text. Today it shows the lowercased
  last message, which can be a path or a progress bar. Map the index verbs to fixed
  phrases:
  - `Updating` and `Fetch` → `updating crates.io index` (Screen B)
  - `Locking` → `resolving dependencies`
  - `Downloading` and `Downloaded` → `downloading crates`
  - anything else, including no message yet → `preparing build`
- Tests (`tests/test_live.py`, plus parser table tests):
  - Add a fake child that replays real
    `cargo install --path . --locked --profile local-install` output, in this order:
    1. `  Installing bob-obsidian v0.4.0 (/repo)`
    2. `    Updating crates.io index`
    3. `\r` `Fetch [` frames
    4. `   Compiling serde …` with `Building [==> ] 87/235: serde` frames
    5. `   Compiling bob-obsidian v0.4.0 (/repo)` and
       `Building […] 253/258: bob-obsidian` frames
    6. `   Finished`local-install` profile [optimized] target(s) in …`
    7. `   Replacing /root/bin/bob`
    8. `    Replaced package …`
    9. `warning: be sure to add …` Space the lines with small sleeps.
  - Assert that the history rows give `crate` the largest share of seconds and give
    `deps` a nonzero share.
  - Assert that the frames show `updating crates.io index`, `compiling dependencies`
    with an `n/m` counter, and `compiling bob-obsidian` in that order.
  - Assert that no `Fetch [`, `Replaced package`, or `Installing bob-obsidian v` text
    reaches scrollback, while the `warning:` line still passes through.

### 2. Two-space indent for every row the runner and digest print (Screens A–E)

- **Runner (`live.py`).**
  - The live frame becomes `  ⠼ compiling …`.
  - The final rows become `  ✓ built · 8m 12s · about usual`,
    `  ✗ build failed after 3:12 · exit 101`, and `  ✗ interrupted after 4:12`.
  - The excerpt lines, `full log  …`, and `rerun with  …` use a 6-space indent (Screen
    D).
  - The frame, indent included, still fits in `min(cols − 1, 100)` cells. Count the
    indent in `rows-file` row math.
  - Raw passthrough lines (rustc warnings and Mac Capture notices) stay byte-for-byte
    unindented.
- **Digest (`digest.py`).** Everything below the banner moves right by 2 cells, and the
  right edge stays at `W = min(COLUMNS − 1, 100)`. So subjects lose 2 cells, and the dim
  SHA is still right-aligned at W.
  - The repo header becomes `  bob-cli · 3 commits since …`, and its notes and wrapped
    label rows move to 4 spaces.
  - Commit rows become `  {badge:2}{type}  {scope}  {subject}…{sha}`, so the badge
    column starts at cell 2: `  ⚠ feat!  …` and `    feat   …`.
  - Bodies, the folded `Verified:` rows, `+ N more …`, and the `git -C … log` line keep
    their positions relative to the commit row (Screens A and E).
  - The narrow-pane rules (drop the SHA below 60 columns, fold the scope below 45) and
    the non-TTY plain mode (Screen F) get the same indent: headers at 2 spaces, subjects
    at 4.
- **Summary replay (`estimate.replay_lines`).** Indent the rows by 2 spaces under the
  summary rows. The plan line is already indented.
- **`scripts/install_all`.** `interrupt_run` prints `  ✗ interrupted after m:ss`.
- Update the goldens in `test_digest.py` (Screen A, Screen E, and the colored snapshot)
  and every assertion in `test_live.py`, `test_estimate.py`, and `test_install_all.py`
  that pins column-0 text.
- Keep the byte-identical `banner()` fixtures as they are.

### 3. Fallbacks that never change the install outcome

- **`scripts/lib/install_ui.sh` `install_ui_run`.** Run the command directly only when
  the runner's status is nonzero, is not 130, and the started file does not exist. A
  runner exit of 0 without a started file returns 0. Add a `test_live.py` case that
  makes the started-file touch fail and asserts the child ran exactly once.
- **`live.run`.** Wrap everything after a successful spawn. On any unexpected exception:
  1. Append the traceback to `crash.log` (`state.append_crash`) and stop drawing.
  2. If the child is still running, copy its remaining output raw to stdout until EOF,
     so it can never block on a full pipe.
  3. Wait for the child, then restore the cursor and signal handlers.
  4. Return the child's exact status. On the interrupt path, keep the SIGINT re-raise.
     An exception before the spawn still returns nonzero without touching the started
     file, so the shim's fallback runs the command once.
- Add PTY tests that inject faults into `history.append_step_rows`, into
  `estimate.estimate` at the final row, and into `PhaseDetector.parse` mid-stream. Use a
  child that floods more than 64 KiB after the fault. Assert that statuses 0 and 101 are
  returned unchanged, the run does not hang, and the child ran once.

### 4. bob-cli summary duration (`scripts/install_all`)

- Set `cli_start="$SECONDS"` immediately before the `🦀 BOB-CLI 1/3` banner, after the
  digest and plan line. The bob-cli summary duration then covers only its build, like
  the other steps (Screen C).
- A self re-exec still restores `SECONDS` from `BOB_INSTALL_ALL_ELAPSED`, so the total
  is unchanged.
- Add an assertion in `test_install_all.py` that the bob-cli summary duration excludes a
  deliberately slow sibling pull, for example a fake `git` delay or a large
  `BOB_INSTALL_ALL_ELAPSED` offset.

### 5. A failure hint that names the command the user ran (`live.py`)

- When `BOB_INSTALL_ALL` is unset, print
  `rerun with  BOB_INSTALL_VERBOSE=1 just install  to stream build output`.
- Under install-all, keep `rerun with  just install-all -v  to stream build output`.
- Test both.

### 6. Rewrite `docs/guides/install-everything.md` (plan docs-acceptance item 1)

Before you edit, read the `docs.md` reference memory with
`sase memory read docs.md -r "<why>"`. Keep the guide's structure and front matter, and
use the guide's example paths (`/home/alex/src/bobs-org`).

- **Sample run.** Replace the stale `installed`/`synced` block with a current sample run
  adapted from Screens A–C: header pull rows, What's new, the `⏱ usually …` plan line,
  one live line, the `✓ built` row, the step row, and the summary.
- **What's new.** It shows commits since each checkout's last successful install on this
  host. Without a watermark it falls back to the pull range. `Verified:` paragraphs are
  folded to one row. Non-terminal runs print plain subjects only.
- **What the ETA phrases mean, and why the first runs only learn.** A range needs 5
  consistent, comparable runs: same host, build kind, toolchain, profile, and
  incremental setting. Cover each phrase:
  - `first timed build on <host>`
  - `last similar build 7m 50s`
  - the bar with `about 2–4 min left`
  - `taking longer than usual (8–10 min)`
  - `Cargo.lock changed, expect a longer build`
  - the `faster than usual`, `about usual`, and `slower than usual` suffixes
- **Escape hatches.** Document `-v`, and add a table of the escape-hatch variables:
  `BOB_INSTALL_UI=0`, `BOB_INSTALL_UI_PYTHON`, `BOB_INSTALL_VERBOSE=1`,
  `BOB_INSTALL_NOTIFY=0`, and `BOB_INSTALL_STATE_DIR`.
- **Log and state locations.** The state directory is
  `${XDG_STATE_HOME:-~/.local/state}/bob-cli/install-all/`. It holds `installed.json`,
  `runs.jsonl`, `logs/<run>-<step>.log` (newest 10 per step), `runs/<run>/`,
  `crash.log`, and `obsidian-restart-pending`.
- **Bell and banner.**
  - The bell rings on a terminal after runs of 60 s or more.
  - On macOS, a banner titled with the final status line posts after runs of 120 s or
    more.
  - Neither fires after Ctrl-C or under `BOB_INSTALL_NOTIFY=0`.
  - Ctrl-C stops the run with exit 130 and skips the later steps.
- **`docs/plugins.md`.** In the install-all paragraph, add one sentence: during
  `just install-all` the sync's diff output goes to the step log, one row per changed
  plugin plus any warnings is shown, and `-v` streams the raw output.
- Run `just docs-check`. If it fails only on checks unrelated to these two pages, do not
  fix them here. Corroborate the existing task (for example `bob-cli-h9` or
  `bob-cli-fo`) through `/sase_new_task` and note it on `bob-cli-i0`.

### 7. Acceptance transcripts (plan docs-acceptance item 3)

Capture both transcripts with `script` on a PTY at `TERM=xterm-256color`. In both runs:

- Use a temp `HOME` with the real `CARGO_HOME` and `RUSTUP_HOME` exported explicitly, so
  the completion install lands in the temp home.
- Use temp `CARGO_INSTALL_ROOT`, `XDG_STATE_HOME`, and `BOB_INSTALL_STATE_DIR`.
- Never let either run touch the default Cargo root, `~/bob`, or the real Obsidian.

The two runs:

- **Real Cargo `just install` from this checkout.** Use a fresh temp `CARGO_TARGET_DIR`
  so the build really compiles: expect about 2–3 min on athena. Confirm:
  - the frames move through `updating crates.io index` → `compiling dependencies n/m` →
    `compiling bob-obsidian` → `installing bob`;
  - the final row is `  ✓ built · …`;
  - no `Fetch [`, `Replaced package`, or opening `Installing` text reaches scrollback;
  - the new `runs.jsonl` rows give `deps` plus `crate` most of the seconds.
- **Sandboxed `scripts/install_all`.**
  - Set up the sandbox: clone this checkout into `<tmp>/bobs-org/bob-cli`, with `origin`
    pointing at a temp bare clone so the header pull stays local. Point `BOB_DIR` at a
    temp vault that holds an empty `.obsidian/`. Leave out sibling checkouts, and give
    `script` a `/dev/null` stdin so clone offers answer no.
  - Reuse the first run's `CARGO_TARGET_DIR` to keep the build short.
  - Before the run, record a bob-cli watermark about 5 commits back, using
    `install_ui mark-installed` on a temp checkout of that commit, or by writing
    `installed.json`. That makes What's new non-empty.
  - Confirm the header pull rows, the digest, the indented live line, the `built` row,
    the summary, and exit status 0.

Attach both transcripts to the epic with
`sase bead note bob-cli-i0 "<one-line summary> @<transcript path>"`.

### 8. Verify

Run all three and fix anything this tale broke:

- `just check-scripts`, which covers the install_ui suite and the portability gate
- `just docs-check`
- `just check`

Do not run `just check-full`.

### 9. Close out epic bob-cli-i0 (final step)

1. Run `sase bead epic-symbols bob-cli-i0`. It listed no entries at planning time.
   Resolve any new entry by wiring the symbol up, making it private, adding a non-test
   pragma, or deleting it, per the Symvision epic-whitelist policy. Re-key an entry only
   when a still-open later bead needs the exemption.
2. Close the epic with `sase bead close bob-cli-i0 --note "<verification>"`. The note
   summarizes:
   - the landing review: all seven phases verified against the plan, and integration
     checked against the non-epic commits;
   - the fixes this tale made (steps 1–6) and the transcripts it attached;
   - the follow-up outcomes already recorded on the epic (`bob-cli-i6`, `bob-cli-i7`,
     `bob-cli-i8`, and the declined workspace note). Never use `--force`. If the close
     is refused, fix the cause and close again.
3. Run `just symvision` if the recipe exists. It did not exist at planning time; if it
   still does not, say so in the close note.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   that `sase bead read bob-cli-i0 -r "<why>"` shows, for
   `plan:202611/install_all_wait_and_whats_new.md`.
5. `bob-cli-i0` has no `parent_bead`, so nothing above it needs closing.

## Out of scope

- macOS verification, build speed, and estimator tuning: the filed tasks `bob-cli-i6`,
  `bob-cli-i7`, and `bob-cli-i8`.
- Changes to bob-plugins or bob-mac-capture.
