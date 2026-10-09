---
tier: tale
title: Repair successor-link edge cases and finish bob-cli-5w landing
goal:
  Successor closes preserve task and ledger edits, match across engines, satisfy the
  read budget, and close bob-cli-5w with verified integration.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-5w.land
bead: bob-cli-5w
status: done
---

- **BEAD:**
  [bob-cli-5w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5w/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-5w.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-5w.land.md)
- **COMMITS:**
  - [fa3d427](https://github.com/bobs-org/bob-plugins/commit/fa3d427de52822c71508e37ac95bbae2e9b1e82a)
    — fix(task-status-cycler): single-pass finalize with rootKey anchors and parity

# Finish successor-links writes, parity, and landing

Implement the remaining work for epic **bob-cli-5w**, then complete its landing in this
same coding turn. The original contract is `plan:202610/successor_links.md`, implemented
in bob-cli and linked bob-plugins; Bob Mac Capture consumes its additive JSON. Read that
artifact with `sase artifact read`, and read this epic's landing/triage notes with
`sase bead read`. This is one bounded repair task in the existing successor engine and
its adapters. No new feature design, separate phases, or successor land agent is needed.

## Audited baseline

The lander read all eleven phase scopes and every note. All phases and prerequisite
tasks bob-cli-5v and bob-cli-3k are closed. No parent bead is listed for bob-cli-5w.
`sase bead epic-symbols bob-cli-5w` was empty. The original plan is still `wip`.

Source reviewed at bob-cli `4cc1281`, bob-plugins `22e96a3`, and Bob Mac Capture
`ee19240`. Epic commits are bob-cli `16df9b6`, `e190a7d`, `eb6fa0d`, `08da012`,
`02029a7`; plugins `aff37aa`, `a06b403`, `fa0631f`, `e8b3584`, `22e96a3`; Mac `f4a36e3`.
The configuration, JSON, notice card, read-time glyph, reopening receipt, status-cycle
hooks, and Swift thin-client presentation exist. The lander's fresh
`cargo build --quiet` passed; five successor/nav/glyph Node suites passed 93/93. Fresh
executable sandbox repros found the failures below despite those green tests.

Intervening changes were inventoried from the first epic commit to HEAD in all three
repos. Capture reset (`db195e3`), override restart (`36df8b8`), and swap (`999816c`,
with grammar/completion companions) preserve tasks rather than complete them. Preserve
that behavior; those operations must not acquire a successor pass. The newer ref-task
work (`e0ba61b`, `e04c421`, `9041927`, `4016229`; plugins `947615d`, `a0a417a`; Mac
`aa47c1f`) moves reading tasks into ordinary parent notes and uses `#ref` as identity.
Successor classification must continue treating them as ordinary tasks. Ref scan/intake
and comma-assist changes use separate paths. Mac `ee19240` already repairs the
override-card compiler problem. Preserve these changes and recheck any later drift
before closing.

## 1. Compose daily task edits with ledger edits

In `src/native/task_complete/successors.rs`, `apply_verdicts` currently constructs
`new_day_text` from the original ledger text separately from `changed_files`. Both
`capture/task_complete.rs` and `capture/pomodoro_close.rs` then stage note changes
followed by `new_day_text`, overwriting status/ID edits when the successor lives in the
day file. Line movement also invalidates its task-block reference.

Repro against the built binary, in an isolated vault with the normal #task statuses:

```markdown
# sase.md

- [ ] #task Predecessor [id:: p] ^p

# 20261005.md

## Pomodoros

- [ ] (**0920-0950**) — BOB
  - [[sase#^p]]

## Tasks

- [?] #task Day job [dependsOn:: p] ^job
```

With `BOB_NOW='2026-10-05 09:30:00'`, `BOB_DIR` and `BOB_DAY_FILE` pointing to that
fixture, `bob capture --format json -- '!sase:p'` reports `[?] -> [*]` and inserts
`[[20261005#^job]]`, but the task stays `[?]`. Remove `^job` and the command panics at
`capture/task_blocks.rs:243` with `tracked task headline is not a task`.

Produce one coherent day post-image containing both task edits and ledger edits. Compute
post-edit task locations by stable block identity, including minted IDs, rather than
stale line positions. Keep the normal batch preimage/rollback contract. Cover both `!`
and Pomodoro close (`=!` or `=x!1`), successors with/without IDs, and task sections
before/after Pomodoros. Assert exact committed bytes, JSON status and task-block lines,
surviving ID/link agreement, no panic, and dry-run equality. Test the analogous plugin
daily-successor path to retain its working editor behavior.

## 2. Preserve embedded-subtask anchors end to end

Rust's `plan_task_complete_item` filters the completed set down to tasks with an
`[id::]`, losing a root that is needed only as an anchor. Keep root/ancestry data
separate from the dependency-ID gate (or retain empty-ID root entries); matching still
uses only nonempty explicit IDs. Do not mint dependency IDs.

Repro: the same planned `sase#^p` root instead contains `- [ ] #task Root ^p` followed
by `  - ![[sub#^s]]`; `sub.md` contains `- [ ] #task Subtask [id:: s] ^s`, and `d.md`
contains `- [?] #task Dependent [dependsOn:: s] ^d`. `!sase:p` closes both tasks but
reports `not_planned_today` and leaves D Ready instead of inserting D after the root
link.

The JS pure planner supports `rootKey`, but real close adapters do not propagate it, and
`normalizeClosedTaskIdentities` in `080-dependencies.js` discards it. Carry the actual
root association from recursive embedded closes through normalization and the queued
successor pass. Do not guess a root from the first closed identity when several roots
close. Test actual Ctrl+Enter and capture paths, with an ID-less root, an ID-bearing
child, multiple roots, and a child that has its own earlier live link. Keep the no-ID
gate and no-recursion-of-dependents rule.

## 3. Treat an incomplete dependency snapshot as unavailable

`capture/dependencies.rs::read_prefiltered_one` converts read and UTF-8 failures to
None, while `DependentsSnapshot::build` sets `available: true` whenever the walk
succeeded. A missing prerequisite can therefore look closed. Repro: add a `bad.md`
containing `- [ ] #task Other [id:: q] ^q\n` followed by byte `0xff`, and make D depend
on `p, q`. Closing P currently reports `checked` and links D.

Distinguish marker-negative bytes (safe to omit) from failed reads and invalid
marker-positive text. Propagate incomplete snapshots through the existing
`unblocked_check: unavailable` contract: the requested close succeeds, successor
recovery/linking is skipped, and D remains unchanged. Cover both close entry points and
a read failure where practical. Preserve the existing no-ID/no-discovery guards and
borrowed staged overlay. A valid staged replacement must not be mistaken for an
unavailable dependency merely because its superseded disk image is unusable.

## 4. Align inbox and link identity between engines

Rust `successors::is_inbox_note_path` recognizes only filenames `mac_inbox.md`. The JS
default recognizes only root `inbox.md`, and `planAndApplySuccessorsNow` does not supply
its richer `isInbox` callback. The actual nav contract is root `inbox.md` plus direct
**area** children whose live frontmatter parent resolves to it. A project or untyped
child does not count. See nav `255-inbox-route.js::classifyInboxNote`,
`655-plugin-inbox-route.js`, and audited `glossary:inbox-file`. Fresh Rust repro on root
inbox emits `inbox:false`.

Use the existing frontmatter/parent resolution conventions in Rust and nav's versioned
inbox API (with an equivalent safe fallback) in the cycler. Read only needed candidate
metadata; retain lazy fast paths. Add shared cases for root, renamed direct area child,
mac/gkeep inboxes with/without the defining metadata, project child, untyped child, and
nested grandchildren.

JS also derives `basenameCounts` only from task-bearing paths in the Tasks cache. With
successor `a/d.md` and unrelated prose-only `b/d.md`, actual Warm finalize inserts
ambiguous `[[d#^d]]`. Rust's walk-only catalog includes both files and requires
`[[a/d#^d]]`. Build JS uniqueness counts from the same eligible Markdown path set as
`capture_dependency_tasks::vault_note_paths`, without reading note bodies. Respect
exclusions and staged/open-editor paths. Correct the inaccurate "task-bearing" wording
in `docs/task-dependencies.md` section 12.4. Test ambiguous prose-only siblings,
excluded directories, case rules, and the day-file exception through the real adapter,
not only a pure helper given precomputed counts.

## 5. Remove duplicate recovery work from the real close

In cycler `130-plugin-references.js::finalizeClosedTasks`, the new recover-and-link pass
is followed unconditionally by legacy `recoverBlockedDependentsNow`. A Warm close still
reads every note; the guard test explicitly bypasses finalize to hide that scan. This is
unfulfilled original epic work (5w.8 proposal #3), not a new optimization task. Use one
gated dependency plan and propagate its recovery counts and failures. Give the
recovery-only cancel API the same Warm cache and editor overlay while retaining Ready
recovery/no-link behavior, with cold fallback.

The lander's real `finalizeClosedTasks` harness with p.md, a/d.md, an open daily editor,
and prose-only decoy.md read decoy twice: once in legacy recovery and once in the
existing reference-retirement scan. Account for the entire actual gesture when enforcing
the warm read budget. Preserve reference retirement semantics; use available link
metadata plus open editors to restrict retirement candidates when that index is
complete, and retain a conservative fallback when unavailable. Do not drop required
retirement writes just to pass a counter.

Update the test to drive the real close/finalize path. Assert no unrelated-note body
reads with complete Warm Tasks/link metadata, correct cold recovery, preimage-error
reporting, one notice, and no duplicate walk advancement. Existing successor, cancel,
retirement, and reopen tests must remain correct. Keep source fragments <=1000 lines,
use generated builds rather than editing main.js, bump the cycler version, and update
its README row.

## 6. Verify fixes and post-start integration

Open other repos only through `/sase_repo`. `sase repo open bob-mac-capture` may fail
because its canonical path is missing; use `sase repo open gh:bobs-org/bob-mac-capture`
as the supported fallback. Read repo instructions.

Add focused integration tests combining successor completion with the newer
restart/swap/reset grammar, including a later item moving the continuation and a later
item consuming/removing the successor. Assert final live-link/net-report truth and that
override/reset alone never complete prerequisites. Include a normal `#task #ref`
successor in a parent note so the intervening ref-task work integrates with the
classifier and link writer.

Run **`just check`**, never `just check-full`, in bob-cli. Run bob-plugins build,
build:check, validate, and the full npm tests after the focused regression suites.
Separate confirmed unrelated failures listed below from epic regressions; do not silence
or fix unrelated tests as part of this tale. Deploy changed plugins with
`bob plugins sync --source <opened-bob-plugins-path>` using the supported option shown
by local help; do not allow bare sync to choose a foreign checkout. Update README
version rows to the manifests while shipping these plugin changes, including the known
concurrent stale rows if still present.

Re-run original release timing acceptance on an isolated copy of the real vault (read
the obsidian reference memory first; never run capture against the live vault). Record
seven samples each for hello, a successful no-completion `=x`, a real ID-bearing `!`
close, and `=x!1`; confirm rows prove actual dependent lookup. Targets are <=40ms /
<=40ms / <=70ms / <=70ms. The early phases reported 130-162ms and phase .11 reported
14-37ms; resolve this discrepancy with the same current binary and full copy rather than
assuming the faster samples cover the same path. If the target fails, finish the bounded
resolver/snapshot hot path within this work. Use `/sase_monitor` for commands that
require a handoff; never wait for this turn's own commit, push, SHA, or future CI to
close the epic.

Mac successor source is already present and its successor suites passed in CI run
37975602555; the unrelated comma-assist failure belongs to bob-cli-60.2. No Swift
dependency computation or new protocol fields are required. Recheck any post-audit Mac
drift for compatibility; reuse existing fixtures and tests for the unchanged wire
contract.

## Follow-up triage already completed by the lander

- All phases' deferred decision-record proposals -> ready memory task **bob-cli-63**.
- All phases' deferred glossary proposals -> ready memory task **bob-cli-64**.
- Nine Pandoc return_links failures (5w.1/.2/.3/.4/.10/.11) -> corroborated
  **bob-cli-5t**.
- Two date-sensitive roll-decay failures (5w.6/.7/.8/.9/.10/.11) -> corroborated
  **bob-cli-5r**.
- Ranker timing flake (5w.8/.10) -> corroborated **bob-cli-3w**.
- Mac comma-assist failure (5w.5/.11) -> issue note on active epic **bob-cli-60**, owned
  by **60.2**.
- README version drift from plugins a0a417a (5w.11) -> issue note on active
  **bob-cli-5y**, caused by **5y.6**; coalesce any rows refreshed while shipping this
  fix.
- Inbox parity (5w.3 #5), Warm legacy scan (5w.8 #3), and the timing shortfalls (5w.3
  #4, 5w.4 #4) are retained as epic work above, so separate task proposals were
  declined. No original proposal is silently dropped.

## 7. Complete bob-cli-5w's landing in this turn

Once the repairs and verification are complete, re-read
`sase bead read bob-cli-5w -r "Verify descendants and parent link before final close"`,
its descendants and any new notes, and the linked original plan via
`sase artifact read plan:202610/successor_links.md "Final linked-plan readiness"`. Check
post-audit commits in every relevant repo. Record exact repairs, tests, timings,
integration, the follow-up outcomes above, and any unrelated failures in the close note.
Ensure all descendant work is complete; if this tale has its own bead below the epic,
close that completed tale normally before its parent (no commit prerequisite).

Run `sase bead epic-symbols bob-cli-5w` and resolve each entry by wiring, privatizing,
adding an appropriate non-test pragma, or deleting it per Symvision policy; re-key only
to a still-open later bead that actually needs the exemption. Then run
`sase bead close bob-cli-5w --note "<concrete verification and triage>"`. If closure
names a leftover exemption or unfinished descendant, finish that work and retry. Never
force a successful landing or force merely to bypass validation. Run `just symvision` if
available after closing (currently bob-cli has no recipe). Open the plans repo with
`/sase_repo`, resolve the original artifact path with
`sase artifact path plan:202610/successor_links.md`, and set **`status: done`** in that
plan's frontmatter. Include every changed repository in `/sase_final`.

Finally read `sase bead read bob-cli-5w -r "Need the parent link"`. The audited epic has
**no parent**, so finish normally. If a parent link was added during drift, follow the
original landing rule: verify and normally close only a parent phase; for a parent plan
audit all descendants/notes, previous landing note, linked plan, and drift, retire its
symbols, close normally, check symvision, mark its plan done, and repeat through fully
complete plan ancestors. Stop and note an incomplete or ambiguous ancestor rather than
forcing it closed.
