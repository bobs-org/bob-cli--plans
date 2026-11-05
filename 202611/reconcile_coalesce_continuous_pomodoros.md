---
tier: tale
title: Coalesce continuous same-type Pomodoros in bob task reconcile
goal:
  bob task reconcile folds back-to-back closed Pomodoro rows of the same type in the
  current daily ledger into one row, without changing any task's credited minutes.
size: medium
decisions:
  attribution_guard:
    ask:
      Should coalescing require every linked task and type to keep exactly the minutes
      Stats credits it today?
    choices:
      exact:
        Only credit-neutral runs join; on 20261105 just 0840-0855 merges, 0820-0840 rows
        stay
      none:
        Every same-type run joins (0820-0855 as one row); Stats re-splits minutes across
        its tasks
    default: exact
    why:
      Keeps per-task minutes honest and reproduces the requested 0840-0855 merge exactly
    answer: exact
  decision_record:
    ask:
      Should the coder add a decisions web record for credit-neutral Pomodoro
      coalescing?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.18l
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 09:54:06
status: wip
---

# Coalesce continuous same-type Pomodoro sessions in `bob task reconcile`

## Goal

Bryan logs small, honest Pomodoros, for example three back-to-back 5-minute `FIX`
sessions. That makes the daily note's `## Pomodoros` section long and noisy.
`bob task reconcile` should fold a run of **closed, back-to-back sessions with the same
name (type)** in the current daily ledger into one closed row. Readability improves, and
nobody is discouraged from logging small sessions.

Today's note, `~/bob/2026/20261105.md`, is the motivating example:

```markdown
- [x] (**0840-0845** [t:: 5m]) — FIX
  - 🍅 [[bob#^off-meets]]
    - apollo research
- [x] (**0845-0850** [t:: 5m]) — FIX
  - 🍅 [[bob#^prompt-for-merge-ids]]
    - apollo agent
- [x] (**0850-0855** [t:: 5m]) — FIX
  - 🍅 [[bob#^fix-x-equals-commas]]
    - apollo agent
```

must become

```markdown
- [x] (**0840-0855** [t:: 15m]) — FIX
  - 🍅 [[bob#^off-meets]]
    - apollo research
  - 🍅 [[bob#^prompt-for-merge-ids]]
    - apollo agent
  - 🍅 [[bob#^fix-x-equals-commas]]
    - apollo agent
```

## Terminology: "coalesce", not "merge"

The glossary already defines **Merged Pomodoro**: a _composite type name_ such as
`BUILD + FIX`, produced by landing, `=*`, and narrowing. To avoid that collision, code,
JSON, human output, and docs call this feature **coalescing** (`coalesce_pomodoros`,
`coalesced_pomodoros`, "coalesced continuous sessions"). Coalescing never creates a `+`
name; it only joins rows whose names are already equal.

## Behavior contract

All rules apply to the **selected current daily ledger** that reconcile already loads
(`native_pomodoro::day_file_for`, honoring `BOB_DAY_FILE`). The pass runs only when that
ledger sits under the vault, the same condition the settle block uses (`daily_relative`
is `Some`). The latest earlier daily note and every other note are never touched.

### C1. Eligible rows

A Pomodoro entry (a column-0 checkbox under `## Pomodoros`, outside fences, as found by
`task_status_hooks::scan_pomodoros`) is eligible only when all of these hold:

- It is **closed**: `native_pomodoro::completed_ledger_task` matches (`[x]` or `[X]`).
  Open rows, including the running timed session and `()` placeholders, never coalesce.
- Its header has exactly the canonical closed shape
  `- [x] (**SSSS-EEEE** [t:: Nm]) — NAME` or `- [x] (**SSSS-EEEE** [off:: Nm]) — NAME`.
  That means four-digit HHMM bounds, start ≤ end (no midnight crossing), one `t` or
  `off` duration written as plain minutes (`\d+m`, the only form writers emit), and a
  non-empty em-dash name tail (`capture_pomodoros::parse_name_tail`) with nothing after
  it. Legacy bare `(**HHMM-HHMM**)` rows, stopwatch metadata, extra fields, a trailing
  `^block-id`, or any other trailing token make the row ineligible, so it is left
  byte-for-byte unchanged.
- Its name is not MEET (`pomodoro_themes::routing::is_meet_name`). MEET rows are tied to
  Event Task clocks and gcal phases and never coalesce, which matches the rule that MEET
  is never merged, split, or narrowed.
- It has at least one direct child (`PomodoroEntry::has_child`). Childless rows are
  already deleted by empty-Pomodoro cleanup.

### C2. Continuous same-type pairs

Two eligible rows A then B can be in the same run when:

1. They are **adjacent in file order**: no other Pomodoro entry and no non-blank
   zero-indent line lies between A's block end and B's header. Only blank lines may
   separate them.
2. They are **continuous**: A's end HHMM equals B's start HHMM exactly. Any gap, even
   one minute, breaks the run. That is why the 20261105 runs end at 0855: 0900 starts a
   new run.
3. They are on the **same clock**: both `[t::]` or both `[off::]`.
4. They have the **same type(s)**: the names' component sequences are equal after
   `plan_budget::split_components` + `normalize_component`, so case and spacing are
   ignored, while order and labels must match. `FIX` and `FIX` coalesce; `FIX` and
   `CHORE` do not; `BUILD + FIX` and `FIX + BUILD` do not.
5. They use the **same direct-child indentation** (`PomodoroEntry::child_indentation`),
   so the joined children stay one sibling list in Obsidian. A tab-indented row never
   absorbs a space-indented one.

### C3. Credit guard (`attribution_guard`)

> [!decision] attribution_guard = exact A run may grow by one more row only if the
> coalesced session would credit every linked task and every name component with exactly
> the minutes the separate sessions credit them today.

Stats (`stats/metrics.rs`, `stats/homes.rs`) and period briefs (`periods/brief.rs`)
split a closed session's minutes evenly across its deduplicated counted Task Links. They
use `pomodoro_homes::split_minutes`, where the remainder goes to the first link.
Per-root, per-project, and per-area minutes are sums of those per-task shares. Theme
minutes are split the same way across the name's components (`resplit_theme_minutes`). A
naive join changes those numbers. For example, `0820-0825` (5m, one link) plus
`0825-0840` (15m, two links split 8/7) would become 20m over three links (8/6/6). The
guard prevents that drift:

- **Task credit.** Add one shared helper in `src/native/pomodoro_homes/session.rs`, next
  to `deduplicated_links`, and re-export it from `pomodoro_homes/mod.rs`. Name it, for
  example,
  `closed_session_task_credit<S: HomeSource>(links: &[SessionLink], minutes: u32, daily_path: &Path, resolver: &HomeResolver<S>) -> Option<BTreeMap<String, u32>>`.
  It uses the **same** dedupe identity as `attribute_session` (`deduplicated_links`) and
  the same `split_minutes` order, and returns task-identity → credited minutes. An
  unlinked session maps one `unlinked` key to all its minutes. It returns `None` when
  any counted link resolves `Ambiguous` or `Missing`, because Stats and briefs treat
  unresolved links differently. A `None` anywhere in the candidate makes the pair
  ineligible.
- Session links come from
  `pomodoro_homes::session::counted_session_links(contents, header_line_index)`, the
  same parser Stats uses, where struck, embedded, and plain dedicated links count. It is
  not re-exported today: add it to the `pub(crate) use session::{…}` list in
  `pomodoro_homes/mod.rs`, or use `prepare_session_scan` +
  `counted_session_links_from_scan`. Session minutes are the row's `[t::]` / `[off::]`
  value (`capture::session_clock`).
- A candidate run passes when the key-wise **sum** of each member's credit map equals
  the credit map of the **coalesced** session. Compute the coalesced session's links by
  running `counted_session_links` over the candidate rewritten text, not by
  hand-concatenating lists, so the guard sees exactly what Stats will see. The same task
  linked in two members dedupes in the coalesced row, and the guard counts that
  correctly.
- **Theme credit.** Because member names are equal sequences of `k` components, the
  theme split is exact iff, for every `j` in `1..=k`, `split_minutes(Σm, j)` equals the
  position-wise sum of `split_minutes(m_i, j)`. Checking every `j ≤ k` covers whatever
  subset of components Stats selects as types without consulting config. For
  single-component names (almost all rows) this always holds.
- Session **counts** do drop. `bob stats` Sessions, Count-measure lead parts, and the
  Day Strip count one session per coalesced row. That is the intended effect of
  coalescing and is documented, not guarded.

> [!decision] attribution_guard = none Skip the credit guard and the `pomodoro_homes`
> helper. Every C1/C2-eligible run coalesces in full. Docs must state that Stats and
> period briefs re-split a coalesced row's minutes evenly across its links, so per-task
> and per-root minutes can shift. Drop the guard-specific tests and keep the other
> tests.

### C4. Run building is greedy, to a fixed point

Walk eligible entries in file order and keep a current run. Extend the run with the next
entry when C2 holds between the run's last row and the next row, and (under `exact`) C3
holds for the extended run as a whole. Otherwise close the run and start a new one at
that entry. Coalesce every closed run of length ≥ 2. If anything coalesced, rescan and
repeat until a pass changes nothing. Entries only decrease, so this terminates.

The fixed point matters for idempotence. A first pass can refuse `R1 + e` yet accept
`R1 + (e + f)` once `e + f` has coalesced. For example, `R1 = 10m{x,y}`, `e = 5m{x}`,
`f = 5m{y}`: `R1 + e` is refused, `e + f` is accepted, and then `R1 + ef` gives x=10,
y=10, which matches. A second `bob task reconcile` run must therefore be a no-op.

### C5. Rewrite

For a run `A₁ … Aₙ`:

- Keep A₁'s header bytes and replace only its parenthetical with
  `(**{A₁.start}-{Aₙ.end}** [{t|off}:: {Σ minutes}m])`. A₁'s status symbol, name
  spelling, and line terminator are preserved.
- Keep A₁'s child lines verbatim, then append each later member's child lines verbatim,
  in order: everything after its header through its block end. Delete the later members'
  header lines and any blank lines that separated the members. Interior blank lines
  inside a member's own child block are kept.
- Preserve the original line terminators (LF/CRLF) of every kept line. Operate on
  `split_inclusive('\n')` segments the way settle does, never on re-joined `lines()`.
- Never touch child bullets' content. `🍅` markers, struck links, and nested notes move
  as-is. Marker repair later in the same run behaves as it does for any closed row.

### C6. Pipeline placement

In `sync_task_statuses` (`src/native/task_status_hooks/sync.rs`), run the pass
**immediately after the settle block** and before the daily `FileScan` contents are
replaced with `normalized_daily`. It then sees the final post-structural,
post-empty-cleanup, post-settle ledger, and every later stage (status derivation,
dependency reconcile, blocked-link consumption, writes) works from the coalesced
contents. After a change, rebuild `updated_lines`, `updated_section`, and
`updated_pomodoro_model` exactly as the settle block does. Reuse a
`HomeResolver::new(DiskHomeSource::new( &request.bob_dir))` for the credit guard.
`--dry-run` runs the same planner and reports, but writes nothing.

### C7. Reporting

- `SyncResult` (`model.rs`) gains `coalesced_pomodoros: Vec<CoalescedPomodoro>`,
  serialized as JSON `coalesced_pomodoros`, always present and `[]` when empty. Each
  item has `name` (A₁'s spelling), `span` (`"0840-0855"`), `clock` (`"worked"` |
  `"off"`), `minutes` (sum), and `sources: [{ "span", "minutes" }]` in order. Do not
  report line numbers: later passes in the same run can shift lines. The field is
  additive, so it is not a breaking JSON change.
- Count the items in the `changed` total and in the human `Summary:` line
  (`{} coalesced`).
- Add a human section after the settled section, in the `print_settled_section` style:

  ```
    coalesced continuous Pomodoros        (dry-run: "would coalesce")
      ⧉ 0840-0855 FIX 15m ← 0840-0845, 0845-0850, 0850-0855
  ```

## Expected result on `2026/20261105.md`

Under `attribution_guard = exact`:

| Rows                                                              | Outcome                            | Reason                                |
| ----------------------------------------------------------------- | ---------------------------------- | ------------------------------------- |
| 0840-0845, 0845-0850, 0850-0855 FIX                               | **coalesce → 0840-0855 [t:: 15m]** | 5+5+5 over 3 links → 5/5/5 matches    |
| 0820-0825 + 0825-0840 FIX                                         | stay separate                      | 20m over 3 links → 8/6/6 ≠ 5/8/7      |
| 0825-0840 + 0840-0855 FIX                                         | stay separate                      | 30m over 5 links → 6 each ≠ 8/7/5/5/5 |
| 0900-0905 FIX + 0905-0910 CHORE                                   | stay separate                      | different types (C2.4)                |
| 0855 → 0900                                                       | run boundary                       | gap                                   |
| 0800-0815 MEET `[off::]`                                          | untouched                          | MEET never coalesces                  |
| 0555-0600 PLAN / 0600-0605 BUILD, 0645-0700 BUILD / 0700-0710 FIX | untouched                          | different types                       |

The request also named 0900-0910. Those two rows are `FIX` and `CHORE`, so the "same
task type(s)" rule (and the vault task "Start auto-merging continuous pomodoros of the
same type!") keeps them apart. This plan follows the stated rule. The implementer must
not special-case it.

## Implementation steps

1. **Credit helper** (`src/native/pomodoro_homes/session.rs`, `mod.rs`): add
   `closed_session_task_credit` as described in C3, with unit tests in
   `pomodoro_homes/tests.rs` using `IndexedHomeSource`. Cover even split, remainder to
   first, dedupe, unlinked, and `None` on Missing/Ambiguous.
2. **Planner** (new `src/native/task_status_hooks/coalesce.rs`, registered in
   `task_status_hooks/mod.rs` next to `settle`): add a pure
   `plan_coalesce<S: HomeSource>(daily_contents: &str, daily_path: &Path, resolver: &mut HomeResolver<S>) -> CoalesceOutcome { contents, coalesced }`
   implementing C1–C5. Reuse the existing helpers: the header parse via
   `native_pomodoro::task_time_range` plus a strict shape check,
   `capture_pomodoros::parse_name_tail`, `split_components` / `normalize_component`,
   `routing::is_meet_name`, `capture::session_clock`, `scan_pomodoros`,
   `counted_session_links`, and `split_minutes`. Add a module doc comment in the style
   of `settle.rs` summarizing the rules.
3. **Wire-up** (`sync.rs`, C6), **model** (`model.rs`: `CoalescedPomodoro` and its
   source item, `SyncResult.coalesced_pomodoros`), and **output** (`output.rs`:
   `changed` count, summary, human section). JSON comes from serde.
4. **CLI help**: in `build_cli()`'s `long_about` (`task_status_hooks/mod.rs`), add one
   paragraph stating the coalescing rule (closed, adjacent, continuous, same clock, same
   name, never MEET, credit-neutral only, idempotent). Then run `just docs-gen` to
   refresh the golden CLI reference and generated docs.
5. **Docs** (`docs/task-status-hooks.md`, the contract page):
   - add a bullet to "In practice it:" next to the settle bullet;
   - add a `### Coalesced continuous sessions` subsection after
     `### Settled off-clock events` that states C1–C5, the Sessions-count consequence,
     and a before/after example;
   - document `coalesced_pomodoros` in the Output section's JSON example and field list,
     where `settled` is described.
   - In `docs/stats.md`, add one sentence where Sessions is defined: a coalesced row
     counts as one session and keeps per-task minutes unchanged (or, under `none`,
     re-splits them).
   - Follow the docs memory routing: contract page plus regenerated reference only, with
     no new guide.
6. **Do not** edit `Cargo.toml`'s version or `CHANGELOG.md`; the release train owns
   them. Use a non-breaking conventional commit such as
   `feat(task-status-hooks): coalesce continuous same-type Pomodoros`.
7. **Decision record** (memory decision `decision_record`):

   > [!decision] decision_record Use `/sase_memory_write` to add one strand to the
   > `decisions` web, e.g. `continuous-sessions-coalesce-credit-neutrally`, in the
   > existing record format: Applies to, Claim, Why with rejected alternatives, Cost,
   > Reopens when, and Evidence. Claim: reconcile coalesces closed, adjacent,
   > continuous, same-clock, same-name, non-MEET rows in the current ledger only when
   > every task and type keeps its credited minutes. Rejected alternatives: a naive join
   > (shifts Stats credit), cross-type joins into `+` names, and a config toggle. Cost:
   > the Sessions count drops, and unequal-length multi-task runs stay separate. Reopens
   > when: per-link minute accounting exists. Evidence: the commit and the
   > `docs/task-status-hooks.md` section. Link `[[glossary:merged-pomodoro]]` to note
   > the naming distinction. Then run `sase memory init`.

   > [!decision] decision_record = no Add no memory. If no human reviewed this plan,
   > file the skipped record as a `memory` task bead via `/sase_new_task`, as the memory
   > skill instructs.

## Tests

Unit tests in `coalesce.rs` (pure, with `IndexedHomeSource` or a small test
`HomeSource`):

- the 20261105 fixture (the Pomodoros section above plus the 0820/0825/0900/0905 rows),
  which yields exactly one coalesced row, `0840-0855 [t:: 15m] — FIX`, with children in
  order and everything else byte-identical;
- different names, a one-minute gap, `[t::]` vs `[off::]`, MEET, an open or running row,
  legacy bare `(**HHMM-HHMM**)`, a trailing `^id`, a midnight-crossing span, and
  child-indentation mismatch: each untouched;
- blank lines between members are dropped, while interior blank lines in a member's
  children are kept;
- CRLF input keeps CRLF;
- the same task in consecutive members dedupes and is accepted when credit matches
  (5m{x} + 5m{x} → 10m{x});
- unlinked + unlinked coalesces, unlinked + linked is refused, and an unresolved link is
  refused;
- multi-component names: `5m PLAN + BUILD` × 2 is refused (theme split 3/2
  - 3/2 ≠ 5/5), while `4m` × 2 is accepted;
- the fixed-point case from C4 coalesces fully in one call, and running the planner on
  its own output is a no-op.

CLI tests in a new `tests/cli/task_status_hooks/coalesce.rs` (register it in
`tests/cli/task_status_hooks/mod.rs`; mirror `settle.rs`, using `--bob-dir`,
`BOB_DAY_FILE`, `BOB_NOW`, `-r 0`):

- `--dry-run --format json` reports `coalesced_pomodoros` and leaves the file unchanged;
- a live run rewrites the ledger, and a second live run reports `[]` and changes
  nothing;
- the human output shows the `⧉` line, or `would coalesce` under `--dry-run`;
- a vault with a linked task note (`bob.md` with `^block` tasks) exercises the real
  `DiskHomeSource` guard path.

## Verification

Run, from the repo:

```bash
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test task_status_hooks
cargo test pomodoro_homes
cargo test --test cli task_status_hooks
just docs-gen
just docs-check
```

Then do a read-only check against the real vault and confirm the reported
`coalesced_pomodoros` matches the table above, with exactly one 0840-0855 FIX item:

```bash
BOB_DAY_FILE="$HOME/bob/2026/20261105.md" cargo run -q -- task reconcile --dry-run --format json -r 0 | jq '.coalesced_pomodoros'
```

Do **not** run a live reconcile against `~/bob` from the branch. The installed `bob`
(cron `10,25,40,55` and plugins) picks this up after the release train ships it.

## Out of scope

- Coalescing across different types into a Merged Pomodoro name (`FIX + CHORE`), or
  across gaps.
- Rewriting earlier daily notes or bulk-migrating history.
- Any bob-plugins or Bob Mac Capture change. Both already render ordinary closed rows.
- A config toggle. Coalescing is always on; the credit guard is the safety rail.
