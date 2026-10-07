---
tier: tale
title: Repair the colliding artifact-link events
goal:
  Make sase artifact doctor healthy, let typed links write again, and backfill the
  relations this outage kept as free text
size: medium
bead: bob-cli-5k.4
proposed_by: bbugyi200.athena.bob-cli-5k.4
create_time: 2026-10-07 15:09:02
status: wip
---

- **PARENT:**
  [202610/close_top_ten_impact_beads.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/close_top_ten_impact_beads.md)
- **BEAD:**
  [bob-cli-5k.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5k/bob-cli-5k.4.md)

# Plan: Repair the colliding artifact-link events

Phase `link-store` of epic bob-cli-5k. Own bob-cli-21. This tale is the repair Bryan
approves before anyone edits the shared plans sidecar. Do the data repair below. Leave
sase source unchanged. Leave the bob-cli checkout unchanged. Do not close epic
bob-cli-5k or any ancestor.

Omit a `links:` frontmatter inlet on every plan proposed while this store is unhealthy.
That inlet archives the scratch plan and then crashes before the approval gate.

## 1. What is wrong

`sase artifact doctor` exits 1. On 2026-10-07 the checkout store reported exactly one
unhealthy cause:

`validation: operation_id de29d2e25c1cfb4381f223c44d576f8c was reused for different artifact link events`

The same run reported `Link event objects 485 durable / 12918 pending (delta +12433)`.
Missing source files in that report are informational and already present. Leave them
alone. `sase artifact doctor --fix` rebuilds projections from durable events and does
not delete event objects, so it cannot repair this.

The reducer (`dedupe_events` in sase-core `artifact_link/events.rs`) accepts one
`operation_id` twice only when the canonical JSON matches. `created_at` is part of that
JSON. A replay that stamps a new `created_at` on the same derived edge becomes a second
content-addressed object and fails the whole store. `assert_healthy` then rejects every
`sase artifact link add` and every plan `links:` inlet.

Doctor names only the first colliding id. A second pair fails the same way as soon as
the first is gone. Both pairs are derived `implements` edges, identical except
`created_at`, written by the same two plans-sidecar commits:

|                                    | Keep (first write)                                                                        | Delete (replay)                                                                           |
| ---------------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Commit                             | `28ee633` at 2026-09-10T13:39:27-04:00                                                    | `c963f9c` at 2026-09-10T14:31:46-04:00                                                    |
| `created_at`                       | `2026-09-10T17:39:25Z`                                                                    | `2026-09-10T18:31:43Z`                                                                    |
| `de29d2e25c1cfb4381f223c44d576f8c` | `link-events/v1/6e/6efae635a11ed1b059bcf2570f6dfcb6bbf69a96f4b7d2e92d5df125f60fe26c.json` | `link-events/v1/51/513bdb2010562a67007a473f61a187bcb41705f6c0e99eb130631f767620c233.json` |
| Edge                               | `plan:202609/capture_task_toggle.md` implements `bead:bob-cli-1z`                         | same edge                                                                                 |
| `bd2e9b172717b0182b4876cd10c6a5c9` | `link-events/v1/74/748c3453ddc3dc3b11b32f4f7dd2b514b6a8528ac82a007f5b68a00983ca5d35.json` | `link-events/v1/27/27ec98b34518ec20fa099e2e83b50f160a34ae1daa9dd89c80e1024b2bfc060f.json` |
| Edge                               | `plan:202609/task_status_groups.md` implements `bead:bob-cli-1y`                          | same edge                                                                                 |

`c963f9c` added only the two delete-column files. Keep the first write because it is the
original fact. Each object alone reduces. Current derivation uses `created_at`
`1970-01-01T00:00:00Z`, producer `sase.artifact-link-derived`, and a different operation
id (the capture-task-toggle edge now hashes to `76970e0839e0059e145f27fdb9904bcb`), so
keeping the historical object does not collide with a later replay of that edge.

The checkout store `sase artifact doctor` reads is the plans sidecar from
`sase repo open plans`, plus the machine-local event root
`~/.sase/projects/gh_bobs-org__bob-cli/link-events` (347 objects, no duplicate operation
ids). On 2026-10-07 that union was the 485 durable objects. The hidden host-owned plans
clone `~/.sase/projects/gh_bobs-org__bob-cli/repos/plans` contains the same four bytes
and was 4 commits behind origin. `sase artifact link add` publishes through
`resolve_machine_artifact_link_store`, which fresh-integrates that hidden clone and
resets it to upstream. A deletion that exists only in a dirty checkout or only as an
unpushed hidden-clone commit is invisible to the next `link add`. The repair commit has
to be on the plans remote before the first `link add`.

Pending events are the machine-local outbox
`~/.sase/projects/gh_bobs-org__bob-cli/artifact-link-outbox.jsonl`, not uncommitted git
files. On 2026-10-07 it held 12918 lines and 1065 unique operation ids: 552 already
matched a durable payload, 511 were not durable yet, and only the two operation ids
above were internally inconsistent. Each of those two had three queued copies, at
`2026-09-10T18:31:43Z`, `2026-09-10T19:01:21Z`, and `2026-09-10T20:01:21Z`. None matched
the kept `17:39:25Z` object. Doctor overlays every queued event on the durable set, so
those six lines keep the store invalid after the git deletion. The remaining queue
reduces cleanly once those two operation ids are absent from the overlay. The large
pending count is an unpublished backlog that piled up while validation refused drains.
It is not a second corruption, and this repair does not drain it.

## 2. Backup, then edit the minimum

Re-scan before editing. Load every `link-events/v1/**/*.json` object under the plans
sidecar `sase repo open plans` prints, group by `operation_id`, and confirm the only
duplicates are the two pairs in the table. If a third operation id is duplicated, stop,
change nothing, and append a `PROPOSED FOLLOW-UP:` on bob-cli-5k.4 naming that id. If
the two delete-column paths are already gone and a fresh doctor exits 0, skip sections 2
and 3 and continue at section 4.

Copy these to `~/.sase/projects/gh_bobs-org__bob-cli/repair-bob-cli-21/` before
rewriting anything:

- the two delete-column JSON files
- the outbox JSONL

Record the plans `HEAD` sha in a `README` in that directory, plus the rollback in
section 6.

In the plans sidecar from `sase repo open plans` only:

- `git rm` the two delete-column paths
- leave the two keep-column paths in place

Leave the hidden plans clone untouched. Its next fresh integrate picks up the pushed
commit.

Rewrite the outbox under an exclusive flock on `artifact-link-outbox.lock`. Write a temp
file in the same directory, fsync, and rename it over the JSONL. Drop a line only when
its event `operation_id` is `de29d2e25c1cfb4381f223c44d576f8c` or
`bd2e9b172717b0182b4876cd10c6a5c9` and its canonical JSON differs from the kept durable
object. Preserve every other line, in order. On the 2026-10-07 measurement that drops
six lines.

Leave `sase artifact doctor --fix`, `prune --apply`, `reclaim --apply`, and trash purge
unrun.

## 3. Commit and push the plans sidecar before any link write

This approved plan instructs the implementer to invoke `/sase_git_commit` on the plans
sidecar.

Run it with the working directory set to the path `sase repo open plans` printed. Pass
`-B keep`. The assigned phase bead stays open until section 5. Message:

```
fix(artifact-links): drop replayed derived link events that reused operation ids

The 2026-09-10 replay commit c963f9c wrote a second content-addressed
object for two derived implements edges that already existed from
28ee633. The edges match; only created_at differs, so the reducer
rejects both operation ids and blocks every later link write.

Keep the first object of each pair. Rollback is reverting this commit.
```

`sase stitch create` pushes on the `create_commit` path. After it exits 0,
`git status --short --branch` in that sidecar must show a clean tree that is not ahead
of upstream. If it is still ahead, push before continuing. An unpushed deletion is not a
finished repair: the next `sase artifact link add` resets the hidden clone to upstream
and restores both colliding objects.

Then run `sase artifact doctor` with no flags. It must exit 0. The pending count should
fall by the lines removed from the outbox (six, unless the re-scan found a different
number) and otherwise stay large. Record the durable count, the pending count, and the
delta in the close note.

## 4. Prove a typed link writes, then backfill

Run each `sase artifact link add` only after section 3 has pushed. Each command must
exit 0. After each one, if `~/.sase/projects/gh_bobs-org__bob-cli/repos/plans` or the
hidden beads clone is ahead of its upstream, push it before the next add. The next add
fresh-integrates those clones and resets unpublished commits back to upstream.

Verification write, which is also the first backfill row:

```
sase artifact link add bead:bob-cli-5i related bead:bob-cli-4j "4j's lib failure masked 5i"
```

`sase artifact link list bead:bob-cli-5i` must show that `related` edge.

Backfill the other free-text `related` edges these notes retained because `link add`
failed. Skip a row that already lists. Use one line of why, taken from the note.

| Source            | Target              | Why                                                                                        |
| ----------------- | ------------------- | ------------------------------------------------------------------------------------------ |
| `bead:bob-cli-5i` | `bead:bob-cli-3c`   | `just check` must run the CLI target so a lib failure cannot mask it                       |
| `bead:bob-cli-5i` | `bead:bob-cli-52`   | bob-cli-52 owns the separate `ingest_characterizes_url_failure_modes` CLI failure          |
| `bead:bob-cli-4t` | `bead:bob-cli-4n`   | adjacent glossary containment work, separate from the inbox answer-routing decision        |
| `bead:bob-cli-4u` | `bead:bob-cli-4s`   | highlights epic bob-cli-4s may edit the same `create.rs`; the Pandoc assertion predates it |
| `bead:bob-cli-58` | `bead:bob-cli-4u`   | both touch the embedded Pandoc filter and flags in `create.rs`                             |
| `bead:bob-cli-5c` | `bead:bob-cli-40`   | same parallel-test interference class as the capture_pomodoros flake                       |
| `bead:bob-cli-5c` | `bead:bob-cli-2e`   | same env-mutation race class                                                               |
| `bead:bob-cli-5g` | `bead:bob-cli-30`   | the Mac crontab bob-cli-30 tracks gains this ref-jobs line                                 |
| `bead:bob-cli-5h` | `bead:bob-cli-4a`   | bob-cli-4a decides the JSON-output flag spelling                                           |
| `bead:bob-cli-24` | `bead:bob-cli-v`    | both are bob-cli lint-gate hygiene                                                         |
| `bead:bob-cli-2t` | `bead:bob-cli-2r`   | close-summary `entry_line` defect found by bob-cli-2r and left outside that epic           |
| `bead:bob-cli-2u` | `bead:bob-cli-2r`   | equal-indent Task Link move found by bob-cli-2r and left outside that epic                 |
| `bead:bob-cli-20` | `bead:bob-cli-1z.5` | proposing phase; its note records the judgment-call capture copy                           |

Search once more with `sase bead search 'typed link' --regex` and add any further note
that names a source, a `related` target, and a rejection by this store. List every
backfilled pair in the bob-cli-21 close note. Leave the original free-text notes in
place.

## 5. Follow-ups, symbols, and close

Append these notes to bob-cli-5k.4 with `sase bead note`:

- `PROPOSED FOLLOW-UP: one bad historical artifact-link event pair must not block every future write — whole-store validation fails every link add while one operation id has two canonical payloads; quarantine that pair instead of rejecting the store.`
- `PROPOSED FOLLOW-UP: sase plan propose must validate or publish the links: inlet before consuming the scratch plan, or roll the archive back on failure — a links: inlet currently archives the plan and crashes before the approval gate.`
- `PROPOSED FOLLOW-UP: drain or compact the bob-cli artifact-link outbox — on 2026-10-07 it held 12918 lines and 511 unpublished operation ids because publication refused while the store was invalid; this repair does not drain them.`

Sase source changes for those three items stay out of this tale.

Run `sase bead epic-symbols bob-cli-5k.4`. On 2026-10-07 it reported no entries. If it
now lists any, re-key each Justfile line to epic bob-cli-5k or to a phase that is still
open. `sase bead close` refuses while leftovers remain.

Close bob-cli-21:

```
sase bead close bob-cli-21 --note "<doctor exit 0 with durable/pending counts; link add of bob-cli-5i related bob-cli-4j listed back; deleted the two c963f9c replay objects and kept the 28ee633 objects; outbox lines removed; every backfilled pair; host>"
```

Then close only the phase bead:

```
sase bead close bob-cli-5k.4 --note "<what doctor and link list showed, and that epic-symbols was clean or re-keyed>"
```

Leave bob-cli-5k open.

## 6. Rollback

Plans sidecar: revert the repair commit and push. That restores the two replay objects.

Outbox: copy `repair-bob-cli-21`'s JSONL backup back over `artifact-link-outbox.jsonl`
under the same flock.

Typed links written in section 4 can stay. They are new operation ids and are not part
of the collision.

## 7. Done

- `sase artifact doctor` exits 0 after the plans commit is on origin.
- `sase artifact link list bead:bob-cli-5i` shows `related` `bead:bob-cli-4j`.
- The backfill table is written, or a row is skipped because it already listed.
- The three follow-up notes are on bob-cli-5k.4.
- bob-cli-21 and bob-cli-5k.4 are closed. bob-cli-5k is open.
- The bob-cli checkout has no edits from this work.
