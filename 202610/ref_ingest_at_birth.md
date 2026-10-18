---
tier: epic
title: bob ref create files its own references (ingest at birth)
goal: 'A successful `bob ref create` on athena, apollo, or the Mac leaves the reference
  in the library — PDF in `lib/`, ref note written, reading task filed — committed
  and pushed as one commit, so `bob ref scan` becomes a repair tool instead of a daily
  step. When inline ingest cannot run, the PDF stays queued in that host''s `xlib/`
  with a visible reason and a per-host `bob ref scan --intake-only` timer files it
  later.

  '
decisions:
  rollout:
    ask: Should the final phase deploy to the tailnet hosts itself (install bob, apply
      chezmoi, enable timers)?
    choices:
      hosts: Deploys to athena, then apollo and the Mac over Tailscale SSH; you do
        only the Mac crontab
      athena: Deploys only on athena and hands you exact apollo and Mac commands to
        run yourself
    default: hosts
    why: Ingest only pays off once every host runs the new bob and the Mac bridge
      retires
    answer: hosts
  decision_record:
    ask: Should the rollout phase record "references are ingested at birth" as a decisions-web
      strand?
    memory:
    - decisions
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    default: true
    answer: true
phases:
- id: scan-sync-lock
  title: Writing scans hold the vault-maintenance lock
  depends_on: []
  size: small
  description: 'scan-sync-lock: make every writing `bob ref scan` (human and JSON)
    take bob_sync.lock before scan.lock and hold both through intake and note writes.
    This closes the window where vault-sync pushes a library PDF without its ref note.
    Add the `lock_busy` error and document the lock order.'
- id: create-ingest
  title: Shared intake session and create wiring
  depends_on: []
  size: medium
  description: 'create-ingest: build the shared locked intake session in a new highlights_ref
    intake module. It takes the locks, pre-syncs, re-checks, moves, plans with rollback,
    writes, makes one scoped commit, and post-syncs. Wire it into every `bob ref create`
    route in place of `next: bob ref scan`. Add `-k/--keep-intake`, the `highlights.ingest_on_create`
    switch, and `BOB_HIGHLIGHTS_INGEST_ON_CREATE`, with scratch-vault integration
    tests.'
- id: producer-ingest
  title: Ingest for listen-attach, ref jobs, and gkeep pull
  depends_on:
  - create-ingest
  size: medium
  description: 'producer-ingest: route the remaining PDF producers through the intake
    session. These are the `--listen` attach path (audio-only units), `ingest_url`
    (the capture ref-jobs worker and `bob gkeep pull`), and their outcome types. Add
    ingested/queued states to `bob ref jobs` and to gkeep output.'
- id: scan-intake-only
  title: bob ref scan --intake-only drain
  depends_on:
  - scan-sync-lock
  - create-ingest
  size: medium
  description: 'scan-intake-only: add `bob ref scan -i/--intake-only`. It drains the
    local xlib/ queue through the intake session, with one commit per reference, and
    skips the annotation pass and the pre-scan hook. It refuses `-w`, adds additive
    JSON fields, and exits non-zero while anything stays queued, so timers surface
    stuck items.'
- id: intake-health
  title: Intake health in bob ref doctor
  depends_on:
  - scan-intake-only
  size: small
  description: 'intake-health: record a per-host intake status file (last success
    and last failure) from create, producers, and scans. Teach `bob ref doctor` to
    show it, plus the age of the oldest queued xlib/ PDF, warning past one hour.'
- id: tailnet-rollout
  title: Per-host drains, bridge retirement, docs, and rollout
  depends_on:
  - scan-sync-lock
  - create-ingest
  - producer-ingest
  - scan-intake-only
  - intake-health
  size: medium
  description: 'tailnet-rollout: in chezmoi, add the Linux bob-ref-intake systemd
    timer, retire the bob_xlib_pull bridge and pre_scan_hook, and fix the maybe_bob_highlights_sync
    gate. Rewrite the Highlights intake docs. Deploy to the hosts per the rollout
    decision, verify, and record the decision strand.'
proposed_by: bbugyi200.athena.research.4s.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 14:40:02
status: wip
bead_id: bob-cli-9c
---

- **PROMPT:** [prompts/202610/ref_ingest_at_birth.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_ingest_at_birth.md)
- **BEAD:** [bob-cli-9c](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9c/README.md)

# Plan: bob ref create files its own references (ingest at birth)

## Context

Today `bob ref create` stops at the intake spool. It renders and stamps
`xlib/<type>/<stem>.pdf`, then prints `next: bob ref scan`. `xlib/` is gitignored. Only
the Mac drains intake: its cron runs the full scan, whose `pre_scan_hook`
(`bob_xlib_pull`) rsyncs athena's and apollo's `xlib/` first. So a reference made on
Linux is unusable — no `lib/` PDF, no ref note, no reading task — until the Mac is awake
and scans. The research measured these numbers:

- **Delay:** median 4.6 min, p90 30 min, maximum 11.6 h with the lid shut.
- **Split commits:** in **24%** of PDF-adding commits, the Mac's vault-sync pushed the
  `lib/` PDF _without_ its note. `scan_library` moves intake before writing notes while
  holding only `scan.lock`, and vault-sync (every 15 s, `bob_sync.lock`) interleaves.

The full research report, with all of its requirements accepted, is
`research:202610/ref_create_auto_ingest_across_tailnet/ref_create_auto_ingest_across_tailnet.md`
(read it with `sase artifact read`). This epic implements its recommended option G,
"ingest at birth". It does **not** automate the full `bob ref scan`. Annotation sync
stays a Mac-only scheduled full scan.

## Requirements (accepted)

1. **Done means filed.** A successful `create` means PDF in `lib/`, ref note written,
   reading task filed, pushed. "Scan runs automatically" is not the goal.
2. **One birth, one commit.** Exactly one host files each reference. The PDF, note, and
   task land in one commit. No commit ever carries a new `lib/` PDF without its note.
3. **Graceful degradation.** If inline ingest cannot run, `create` still exits 0 and
   prints `queued: … (<reason>)`, and a local drain finishes the job later. The Mac is
   never required for intake.
4. **Every producer.** This covers `create` (all routes plus `--listen` attach), the
   capture ref-jobs worker, and `bob gkeep pull`.
5. **Annotation sync unchanged.** New-reference ingest never writes PDF bytes
   (`write_pdf = false`); create already stamped the marker.
6. **Contract change.** `create` on athena or apollo now makes vault commits, including
   SASE agents' creates. `-k/--keep-intake`, `highlights.ingest_on_create: false`, and
   `BOB_HIGHLIGHTS_INGEST_ON_CREATE=0` restore today's behavior byte for byte.
7. **Honest offline bound.** A host that cannot pull first queues; it never ingests
   blind. `bob ref scan` still ingests locally on demand.

## Shared design (all phases follow it)

### Lock order

The lock order, documented everywhere, is: **`ingest.lock` (ref-jobs only) →
`bob_sync.lock` → `scan.lock`.**

- `bob ref rename` already takes `bob_sync.lock` then `scan.lock`. The intake session
  and (after `scan-sync-lock`) every writing scan use the same order.
- `ingest_url` keeps holding its own `ingest.lock` while it calls the session. Nothing
  takes `bob_sync.lock` before `ingest.lock`: `bob gkeep pull` clips in
  `run_clip_pre_pass` _before_ it takes the vault lock. Verify this stays true.
- Never acquire `bob_sync.lock` twice in one process. fs2/flock blocks on a second open
  file description, so a nested acquire would time out against itself.

### The intake session (owned by `create-ingest`, reused by later phases)

Add a new module `src/native/highlights_ref/intake.rs`. The names below are indicative;
the shape is not.

- `IntakeUnit` has three variants:
  - `Pdf { source: xlib PDF, destination: lib PDF }`, with companions (same-stem `.md`,
    `.textbundle`, `mp3/m4a/ogg/opus`) collected by a **single-source** planner. Extract
    the per-source body of `plan_xlib_intake` (`doctor.rs`) and
    `intake_companion_moves`. Unrelated xlib files never abort it.
  - `LibraryPdf { pdf }`, for `-o lib/...` creates. The PDF is already in `lib/`.
  - `Audio { source: xlib audio, destination: lib audio }`, for late audio whose lib PDF
    exists, mirroring `plan_xlib_intake`'s second loop.
- `IntakeSession::open(config, budgets, progress)` does the following. Every failure
  becomes a `QueueReason`; nothing has been touched yet.
  1. **Vault identity.** The canonicalized `config.bob_dir` must equal the canonicalized
     `bob_env::bob_dir()`. vault-sync always syncs `BOB_DIR`, so a `-b` vault that
     differs must never be committed or synced through it (`other_vault`). Also,
     `lib_dir`, `ref_dir`, and `xlib_dir` must resolve inside the vault
     (`outside_vault`).
  2. `ob::acquire_lock_waiting(60 s, on_first_wait)`. The first wait reports
     `waiting for another vault maintenance run…` through the progress callback. On
     timeout: `lock_busy`.
  3. `ob::detect_git_worktree`. If the vault is not a worktree: `not_a_worktree`.
  4. `acquire_scan_writer_lock_with_timeout(30 s)`. On timeout: `scan_busy`.
  5. The internal env `BOB_REF_INTAKE_LOCK_WAIT_SECONDS` overrides both waits (tests use
     `0`).
- `pre_sync()` calls
  `vault_sync::run_cycle_with_existing_lock_report(&child_env, true)`. `!ok` or any
  conflict → `pre_sync_failed` with the detail, exactly as `rename/plan.rs` judges it.
  **Skip the pre-sync for `LibraryPdf`**: its PDF already sits in the tracked tree, so a
  pre-sync would commit the PDF alone.
- `ingest(unit, producer)` runs per unit:
  1. **Re-check against the freshly pulled tree.**
     - For `Pdf`: the lib PDF, every lib companion destination, the lib sidecars
       (`library_destination_sidecars`), stale lib audio, and the ref note path must all
       be absent; otherwise `collision`. The stem was chosen before a render that can
       take minutes. A stem taken meanwhile queues; it is never renamed.
     - For `Audio`: the lib audio must be absent and the lib PDF present.
  2. **Size.** Every file to be moved must be under vault-sync's 95 MiB limit; otherwise
     `too_large`. Expose `LARGE_FILE_LIMIT_BYTES`, or a `pub(crate)` predicate, from
     `vault_sync.rs` instead of duplicating it.
  3. **Move, then plan, with rollback.** `plan_pdf_sync` reads the PDF at its path and
     derives `source_pdf` and the sidecar from it, so it must see the library location.
     - Move the PDF and companions into `lib/` with no-replace renames. Hoist
       `rename_no_replace` from `rename/plan.rs` into a shared helper.
     - Then plan exactly as `sync_pdf` does: `ScanContext::build`, `plan_pdf_sync` with
       `SyncOptions { dry_run: false, write_pdf: false, prefer: None }`,
       `finalize_annotation_task_plans`, `reserve_preview_ids`, then
       `ensure_safe_to_write`. Extract a non-printing single-PDF plan/write pair from
       `sync_pdf` rather than duplicating its sequence.
     - If planning or the guard refuses, move everything back with no-replace renames
       and queue: `refused`, or `needs_write_pdfs` when the refusal is the "requires
       --write-pdf" one.
     - `bob_sync.lock` is held throughout, so vault-sync never observes the intermediate
       state.
  4. **Write.** Call `execute_pdf_sync`: note plus reading task, never PDF bytes.
     - On error, if `git status --porcelain` shows nothing changed besides the moved
       files, roll the moves back and queue `refused`. The tree is clean after the
       pre-sync, so this check is reliable.
     - Otherwise return `Incomplete { error }`. Do not commit; the next full scan adopts
       the inserted task (existing v2 adoption).
  5. **Scoped commit.** Build the path set from the plan and its write report:
     - every moved lib file;
     - the note path;
     - `routed_task_note_writes`;
     - `routed_intents`;
     - `image_assets`;
     - the reading-task destination file.

     Factor the guard's touched-path walk in `guard.rs` into a helper that also returns
     the reading-task and V2-default destinations it exempts from the veto. Drop paths
     that `git check-ignore` reports and warn about them; `git add` of an ignored path
     fails. Then call `ob::commit_paths` with subject
     `<producer command>: ingest <type>/<stem>`, where the producer is `bob ref create`,
     `bob ref scan`, `bob ref jobs run`, or `bob gkeep pull`. Audio units use
     `<producer command>: attach audio <type>/<stem>`.
     - If the commit fails, report "ingested; commit failed (…); background vault-sync
       will commit it". Every file is written and the lock is held until the end, so
       vault-sync still commits them together.

- `post_sync()` runs one more cycle.
  - On failure, report `push pending (<detail>)`. Exit 0, matching rename.
  - On conflicts, report them (`see _conflicts/sync_conflicts.md`).
- Dropping the session releases `scan.lock`, then `bob_sync.lock`.
- `finish_intake(config, unit, request) -> IntakeOutcome` is the one-unit wrapper that
  create and the producers call. It runs `open`, `pre_sync`, `ingest`, `post_sync`.
  `request` carries the `producer`, a `keep` flag (`-k`), `dry_run`, and an optional
  progress callback.
  - It **never prints**: `ingest_url` promises silence on stdout and stderr.
  - Before anything else it applies the kill switch and `-k`, returning `Kept { why }`.

`IntakeOutcome` has five variants:

| Variant                     | Contents                                                                                                                                        |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `Ingested`                  | lib PDF, note path, note action, reading destination and block id, whether adopted, moved audio, commit sha or commit warning, post-sync status |
| `Queued`                    | xlib path, `QueueReason`, detail                                                                                                                |
| `Kept`                      | why: `-k`, config, or env                                                                                                                       |
| `Incomplete`                | error                                                                                                                                           |
| `WouldIngest` / `WouldKeep` | dry run only                                                                                                                                    |

`QueueReason` slugs are stable for later JSON (bead `bob-cli-5h`): `lock_busy`,
`scan_busy`, `other_vault`, `outside_vault`, `not_a_worktree`, `pre_sync_failed`,
`collision`, `too_large`, `refused`, `needs_write_pdfs`.

### Human output for create (exact shapes)

| Outcome                                | Output                                                                                                                                                                                                                                                                                                                   |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Ingested                               | `ingested: lib/<type>/<stem>.pdf → ref/<type>/<stem>.md · task → <dest>.md ^<block>`, with ` (adopted)` appended when adopted. Then `sync: committed <sha7> · pushed`, or `· push pending (<detail>); background vault-sync will publish it`, or `· pushed with conflicts in <paths> (see _conflicts/sync_conflicts.md)` |
| Queued                                 | `queued: xlib/<type>/<stem>.pdf (<reason detail>) · next: bob ref scan`. Exit 0                                                                                                                                                                                                                                          |
| Kept                                   | `next: bob ref scan`, byte-identical to today                                                                                                                                                                                                                                                                            |
| Incomplete                             | stderr `error: ingest stopped after moving the PDF into lib/: <error>`, stdout `next: bob ref scan`. Exit 1                                                                                                                                                                                                              |
| Dry run                                | `intake: would ingest into lib/<type>/<stem>.pdf → ref/<type>/<stem>.md (pull, file note and reading task, commit, push)` or `intake: keep (<why>)`, then the existing `writes: none`. No locks, no sync                                                                                                                 |
| `External` (`-o` outside lib and xlib) | today's `scan:` and `next: bob ref sync …` lines, unchanged                                                                                                                                                                                                                                                              |

Style the new lines like the surrounding create output.

### Kill switch

`highlights.ingest_on_create: true|false` (default `true`) is parsed in
`parse_highlights_config` (`src/native/config/mod.rs`, `RawHighlights`). The env
`BOB_HIGHLIGHTS_INGEST_ON_CREATE` wins over it:

- `0/false/no/off` disables; `1/true/yes/on` enables (case-insensitive).
- Blank means unset.
- Any other value is an error naming the variable and the accepted values.

Follow `listen.rs` precedence and `diagrams.rs` value parsing. `create`, the ref-jobs
worker, and `gkeep pull` all honor it. `bob ref scan -i` ignores it, because it is an
explicit request.

## Phase scan-sync-lock: Writing scans hold the vault-maintenance lock

Independent of every other phase. It closes the 24% split-commit window on whichever
host scans (the Mac's cron and ⌘S).

1. In `scan_library` (`sync.rs`) and `scan_library_json` (`scan_json.rs`), a non-dry
   scan first calls `ob::acquire_lock_waiting(<budget>, on_first_wait)`, then the
   existing `acquire_scan_writer_lock`, and holds both until return.
   - The hook, intake moves, planning, and every note write happen under both locks. Dry
     runs take neither lock.
   - The budget is 120 s, like the scan lock. The internal
     `BOB_REF_SCAN_LOCK_WAIT_SECONDS` overrides both waits.
   - The first wait prints `waiting for another vault maintenance run…` once to stderr
     (human and JSON modes).
2. On timeout, fail with a new error code `lock_busy`. The message names the maintenance
   lock path. The hint says vault-sync, nightly, or rename held it and to rerun. Add the
   code to `scan_error_parts` and to the JSON error envelope.
3. A `pre_scan_hook` now runs under `bob_sync.lock`. A hook that itself runs
   `bob vault-sync` becomes a silent no-op (vault-sync skips a held lock). Document
   that.
4. Do not add syncing to scans. vault-sync's next cycle after release commits the PDF
   and note together.
5. Tests (`tests/cli/highlights/scan.rs` style, `hold_scan_writer_lock` as the model):
   - Hold `bob_sync.lock` with fs2 on an explicit `BOB_VAULT_SYNC_LOCK_FILE`, with wait
     `0`. The human scan fails naming the lock; JSON emits `lock_busy` in the documented
     envelope.
   - A dry run ignores the held lock.
   - A writing scan with the lock free behaves exactly as before.
6. Docs:
   - `docs/highlights-ref-sync.md`:
     - the "Writer lock" section describes both locks and the order;
     - the error-code table gains `lock_busy`;
     - the "Scan, Safety, and Git/ob Behavior" paragraph says writing scans hold the
       maintenance lock but never sync.
   - `docs/vault-git-sync.md`: the `bob_sync.lock` sharing list adds writing
     `bob ref scan` and `bob ref rename` (already true, undocumented).

Done when: `just check` passes and the docs state the lock order
`bob_sync.lock → scan.lock`.

## Phase create-ingest: Shared intake session and create wiring

1. Build `intake.rs` exactly as the shared design above describes. Include the
   single-source intake planner, the hoisted no-replace rename, the guard path-set
   helper, the non-printing single-PDF plan/write pair from `sync_pdf`, and the
   vault-sync size predicate.
2. Add the kill switch (config key and env) with config parse tests.
3. Wire create. Replace `print_next_step` at every create site with a helper that maps
   the target plan to an `IntakeUnit`, calls `finish_intake`, and prints the outcome:
   - `create.rs` snapshot route (~1357);
   - Markdown / `-M` (~2012);
   - local PDF (~2213);
   - PDF URL (~2410);
   - arXiv (~2666);
   - `clip.rs` web article (~637);
   - `collection.rs` multi-URL (~974).

   The workflow mapping is `Intake` → `Pdf` unit and `Library` → `LibraryPdf`;
   `External` keeps today's lines. The Markdown dry-run `print_plan` (~3417) and every
   route's dry-run printer gain the `intake:` line. Keep `print_next_step` for
   `External` and the kept path.

4. Add `-k, --keep-intake` to `bob ref create`; `-k` is free. Help text: "Leave the new
   PDF queued in xlib/ for a later bob ref scan instead of filing it into the library
   now". Keep options alphabetical (read `cli_rules.md` with `/sase_memory_read` first).
   Rewrite the after-help passages that promise a later scan:
   - "Web articles";
   - "Output" ("Scan moves intake PDFs…");
   - the `-N` help ("the ref note bob ref scan writes").
5. Leave `attach.rs`, `ingest_url`, and scan untouched; later phases own them.
6. Tests: add a new `tests/cli/highlights/ingest.rs` and register it in
   `tests/cli/highlights/mod.rs`. Use `init_vault_sync_pair`, and the fastest local
   route (a local Markdown file or local PDF).
   - Default create:
     - the PDF is in `lib/`, the note is written, the reading task is in its parent, and
       `xlib/` holds no PDF;
     - origin has exactly one new commit, with the subject above, touching exactly the
       PDF, note, and parent;
     - `git status --porcelain` is empty, which proves the path set misses nothing;
     - the peer clone pulls all three;
     - a follow-up `bob ref scan --dry-run` plans no note create.
   - `--listen` / `-a` audio companion: the mp3 moves with the PDF in the same commit.
   - `bob_sync.lock` held, wait `0` → `queued: … (lock busy…)`, exit 0, PDF intact in
     xlib, no commit.
   - Scan lock held → `scan_busy` queue.
   - A git vault whose origin is unreachable → `pre_sync_failed` queue; a later
     `bob ref scan` ingests it locally.
   - Non-git vault → `not_a_worktree` queue.
   - `-b` differing from `BOB_DIR` → `other_vault` queue; the `BOB_DIR` vault is
     untouched.
   - Planning refusal → rollback: seed two orphan reading tasks for the same ref note so
     birth-task selection refuses. The PDF ends back in xlib, the queue reason is
     `refused`, and the tree is clean.
   - `-k`, config `false`, env `0` → output byte-identical to today, no lock, no commit.
   - `--dry-run` → the `intake:` line plus `writes: none`, no lock taken.
   - Unit tests for the re-check (`collision` for an occupied lib PDF, lib sidecar, lib
     audio, or ref note) and for `too_large` using a sparse file.
   - Update existing create assertions whose output changed. Most create tests use
     non-git vaults and now print a `queued: … · next: bob ref scan` line; `contains`
     checks on `next: bob ref scan` keep passing.
7. Docs:
   - `docs/highlights-create.md`: the "next: bob ref scan" passages (~616, ~654), the
     new intake behavior, `-k`, and the switch.
   - `docs/highlights-ref-sync.md`: the config key and env; replace the "`bob ref` does
     not reconcile the vault before or after writes" sentence with the create and intake
     behavior.
   - `docs/ref.md` "Choose a command": create now files; scan is repair and annotation
     sync.

Done when: `just check` passes and every acceptance test above exists and passes.

## Phase producer-ingest: Ingest for listen-attach, ref jobs, and gkeep pull

1. **Attach.** The `--listen` on an existing capture path (`attach.rs` ~382) currently
   prints a literal `next: bob ref scan` after writing `xlib/<rel>.mp3`. Replace it:
   - If the capture's PDF is in `lib/`, file an `Audio` unit: move the mp3, re-sync the
     existing note so it gains the `audio:` frontmatter and embed, and commit the mp3
     plus the note.
   - If the capture's PDF is still queued in `xlib/`, file its `Pdf` unit; the mp3 rides
     along as a companion.
   - `-k` keeps today's output.
2. **`ingest_url`** (`ingest.rs`):
   - After a successful clip, still holding `ingest.lock`, call `finish_intake` with the
     producer taken from a new `IngestRequest` field: `bob ref jobs run` or
     `bob gkeep pull`. Honor the kill switch. Pass `request.progress` as the progress
     callback, so nothing prints.
   - `IngestOutcome::Created` gains an intake field (ingested with note and reading
     destination, or queued with reason), and its `pdf` becomes the final path.
   - `AlreadyQueued` / `AlreadyInLibrary` dedupe is unchanged.
3. **Ref jobs** (`ref_jobs/`):
   - `DoneRecord` gains optional `intake` (`ingested|queued`) and `note`. Old records
     without them still read as today.
   - `JobState` gains `Ingested`; `Clipped` now means clipped-but-queued. Give both
     clear labels and glyphs.
   - The worker line becomes `✓ filed <display> → ref/…md` or
     `✓ clipped <display> → xlib/… (queued: <reason>)`.
   - The fixed footer at `ref_jobs/output.rs` ~151 ("the Mac runs it every 15 minutes")
     shows only when queued rows exist, reworded to the per-host `bob ref scan -i`
     drain.
   - JSON summary keys stay additive.
4. **gkeep pull** (`gkeep/pull.rs`):
   - `ClipOutcome::Created` carries the note and intake.
   - The human line says filed vs queued.
   - JSON adds `note` and `intake` keys additively.
   - The journal's `pdf` records the final path. Confirm no journal reader depends on
     the xlib location; the planner keys on `(id, fp)`.
5. Leave the capture `in_intake` verdict wording ("waiting for bob ref scan") unchanged.
   Bob Mac Capture mirrors it verbatim, and it stays accurate for queued PDFs.
6. Tests:
   - Attach to a library capture → audio filed and committed with the note.
   - Attach to a queued capture → both filed.
   - The ref-jobs worker with the fake web-clip adapter in an `init_vault_sync_pair`
     vault → the job reads `ingested`, a commit lands, and `bob ref jobs` shows it.
   - The same with lock held → `clipped` / queued with reason.
   - gkeep pull clip outcome and JSON keys (existing gkeep test harness).
   - Old done records render as before.
7. Docs: `docs/ref-jobs.md` lifecycle (no Mac wait), layout (lock order), and doctor;
   `docs/gkeep.md` clip outcome; `docs/highlights-create.md` attach.

Done when: `just check` passes and no producer prints or documents "the Mac runs bob ref
scan" as the normal path.

## Phase scan-intake-only: bob ref scan --intake-only drain

1. Add `-i, --intake-only` to `with_scan_args` (`cli.rs`); `-i` is free. It conflicts
   with `-w/--write-pdfs` (clap usage error, exit 2). Help: "File queued intake PDFs and
   audio from xlib/ into the library (pull, ref note, reading task, commit, push); skips
   the annotation pass and the pre-scan hook".
2. Flow in a new `scan_intake_only` (human and JSON paths):
   1. Validate the layout.
   2. Enumerate units without locks: every xlib PDF as a `Pdf` unit with companions, and
      every xlib audio whose lib PDF exists as an `Audio` unit, sorted.
   3. **Empty queue:** print `intake: nothing queued` and exit 0. Take no locks, run no
      sync, so idle timers cost nothing.
   4. `--dry-run`: list `would ingest …` per unit and exit 0, no locks.
   5. Otherwise call `IntakeSession::open`, then `pre_sync`. Either failing is a hard
      error (`lock_busy`, `scan_busy`, `not_a_worktree`, `other_vault`,
      `pre_sync_failed`), exit 1, nothing moved.
   6. `ingest` each unit with producer `bob ref scan`, one commit per reference. A
      queued unit reports `queued: <xlib path> (<reason>)` and the drain continues.
   7. `post_sync` once (a failure warns).
   8. The summary line counts filed and queued units.
   9. Exit 1 if any unit stayed queued or was incomplete, so a stuck item fails its
      timer run visibly; otherwise exit 0.
3. Never run `pre_scan_hook`; JSON `hook.status` is `"skipped"`.
4. JSON stays `schema_version: 1`. Bob Mac Capture decodes tolerantly, so append keys
   **at the end** of both envelopes:
   - `intake_only` (bool);
   - `queued` (`[{pdf, reason, message}]`; normal scans emit `false` / `[]`);
   - plus error code `pre_sync_failed`.

   `intake` keeps listing completed PDF moves and `notes` lists filed notes. Update the
   key-order tests.

5. Tests:
   - A queued PDF (made with `create -k`) drains into one commit on origin.
   - Two queued PDFs → two commits.
   - A stem taken by a peer push between queueing and draining → `collision` queued,
     exit 1, nothing overwritten.
   - Empty queue → no lock (prove by holding `bob_sync.lock`), exit 0.
   - `-i -w` → usage error.
   - `--dry-run` and JSON envelopes (key order, `hook.status`).
   - Late audio unit.
   - A lib PDF with no note is _not_ touched by `-i`.
6. Docs: `docs/highlights-ref-sync.md` — the flag, the flow, the JSON additions and
   error code, and the "typical order" (create files; `-i` drains; the full scan is
   annotation sync and repair).

Done when: `just check` passes and `bob ref scan -i --help` reads cleanly.

## Phase intake-health: Intake health in bob ref doctor

1. Per-host status record at `bob_cli_state_dir()/ref/intake-status.json`, written
   atomically:

   ```json
   {
     "schema_version": 1,
     "last_success": { "at": "…", "command": "…", "filed": 0 },
     "last_failure": { "at": "…", "command": "…", "reason": "…", "message": "…" }
   }
   ```

   - Writers: `finish_intake` (create, attach, ref jobs, gkeep), `bob ref scan -i`, and
     every writing full scan.
   - A queue reason counts as a failure.
   - A failure to write the record only warns; it never changes a command's result.
   - Tests isolate it through the harness's `XDG_STATE_HOME`.

2. `bob ref doctor` (`doctor.rs`, next to `xlib_pending`):
   - `xlib_pending: N` gains ` · oldest <age>` when N > 0. The age is computed from file
     mtimes; reuse `age_short_secs` from `ref_jobs/doctor.rs`.
   - Past 3600 s, warn:
     `xlib intake: N queued, oldest <age> — run bob ref scan -i (the bob-ref-intake timer drains it on athena and apollo)`.
   - Add an `intake_last:` row: `ok <age> ago (<command>, <n> filed)` and
     `failure <age> ago (<reason>: <message>)`, or `none recorded`.
   - Keep `xlib_pending: 0` byte-identical.
3. Tests: doctor rows with a fresh vs >1 h-old queued PDF (set the mtime), with and
   without a status record. Writers update the record on success and on a queue.
4. Docs: the doctor section in `docs/highlights-ref-sync.md`.

Done when: `just check` passes.

## Phase tailnet-rollout: Per-host drains, bridge retirement, docs, and rollout

Open the `chezmoi` linked repo with `/sase_repo`, read its `AGENTS.md`, and follow its
rule to run `chezmoi update -a --force` after committing.

1. **chezmoi changes** (one commit):
   - `home/dot_config/systemd/user/bob-ref-intake.service`:
     - `Type=oneshot`;
     - `Environment=` `HOME`, `BOB_DIR`, and `PATH` exactly as `bob-vault-sync.service`;
     - `ExecStart=/home/bryan/.cargo/bin/bob ref scan --intake-only`;
     - `Nice=10`.

     The systemd user manager sets `XDG_RUNTIME_DIR`, so the unit shares vault-sync's
     `/run/user/1000/bob_sync.lock`.

   - `bob-ref-intake.timer`: `OnBootSec=2min`, `OnUnitActiveSec=10min`,
     `Persistent=true`. Use a timer, never a `.path` unit.
   - A `run_onchange_after_` script that, on athena and apollo only, runs
     `systemctl --user daemon-reload && systemctl --user enable --now bob-ref-intake.timer`.
   - `.chezmoiignore` entries that keep the units off the Mac.
   - Retire the bridge:
     - Remove `pre_scan_hook` from `home/dot_config/bob/config.yml`, leaving a one-line
       comment saying each host drains its own `xlib/`.
     - Delete `home/bin/executable_bob_xlib_pull` and
       `tests/bash/bob_xlib_pull_test.sh`, and add `bin/bob_xlib_pull` to
       `.chezmoiremove` so hosts drop the installed copy.
     - Rule: **each host's `xlib/` has exactly one drainer.** A cross-host rsync racing
       a local drain births two notes.
   - Fix the `home/bin/executable_maybe_bob_highlights_sync` gate. Without a hook it
     currently scans only when `lib/` changed today. It must also scan when `xlib/`
     holds any file, so the Mac's own queued creates drain. Update the matching
     `tests/bash` tests and run them.

2. **Deploy** (per the `rollout` decision). Work in this order, so the window where a
   new Linux create races the old Mac bridge stays minutes long. That race needs an
   rsync to start inside a seconds-long install-to-move window.

   > [!decision] rollout = hosts
   >
   > 1. Install the merged bob on athena: `just install` from an up-to-date master
   >    checkout.
   > 2. Over Tailscale SSH, on the Mac (when awake) and on apollo, pull the bob-cli
   >    checkout (discover its path; Bryan's checkouts live under
   >    `~/projects/github/bobs-org/`) and run `git pull --ff-only && just install`.
   >    Never use `install-all`, which redeploys plugins and restarts Obsidian.
   > 3. Commit and push chezmoi, then `chezmoi update -a --force` on the Mac first, then
   >    apollo, then athena.
   > 4. A host that is unreachable is skipped and reported with its exact commands.

   > [!decision] rollout = athena Install bob and apply chezmoi on athena only, then
   > report the exact apollo and Mac commands (bob install, `chezmoi update -a --force`,
   > verification) for Bryan to run.

3. **Verify** on every deployed host:
   - `bob ref create --help` lists `-k`;
   - `bob ref scan -i --dry-run` runs;
   - `bob ref doctor` shows the intake rows;
   - on Linux, `systemctl --user list-timers bob-ref-intake.timer` and the last run's
     journal;
   - `bob_xlib_pull` is gone and config has no `pre_scan_hook`.

   Do **not** create throwaway references in the live vault. If a real create happened
   after deploy, cite its `ingest` commit as end-to-end evidence; otherwise say the
   first real create is the live check.

4. **Mac crontab.** macOS refuses `crontab` writes over SSH (bead `bob-cli-30`). Do not
   attempt it. In the final report, give Bryan the exact staggered block from
   `docs/vault-git-sync.md` "Mac scheduled maintenance" to install from a Mac Terminal:
   - `2>&1` on the highlights line;
   - `--retry-timeout 120` on reconcile.
5. **Docs** (bob-cli):
   - Rewrite `docs/vault-git-sync.md` "Highlights bridge" as "Highlights intake":
     - ingest at birth under the lock order;
     - per-host `bob-ref-intake.timer` drains;
     - the single-drainer rule;
     - the Mac's full scan is annotation sync plus its own queue;
     - `bob_xlib_pull` is retired;
     - recovery: `ssh <host> bob ref scan -i`.
   - Update `docs/ref-jobs.md` "scan happens later via the Mac cron" if anything
     remains.
   - Grep `docs/` and `README.md` for `bob_xlib_pull`, `pre_scan_hook: PATH`, and "next:
     bob ref scan" and fix stale passages.
6. **Memory.**

   > [!decision] decision_record Use `/sase_memory_write` to add a decisions-web strand,
   > keyword "References Are Ingested At Birth":
   >
   > - **Applies to:** bob-cli and the vault hosts.
   > - **Claim:** the host that renders a reference files it under
   >   `bob_sync.lock → scan.lock` with pre-sync and one scoped commit, and queues
   >   locally otherwise; annotation sync stays the Mac's full scan.
   > - **Why:** this research ref, the 24% split-commit and 4.6 min/11.6 h delay
   >   measurements, and the rejected options A–F.
   > - **Cost:** agents' creates commit to the vault; parent-note merges rely on git.
   > - **Reopens when:** `_conflicts/sync_conflicts.md` shows same-stem collisions
   >   (revisit the single-publisher design).
   >
   > Cite the merged commits. Then run `sase memory init`.

Done when: chezmoi is committed and applied, every reachable host is verified, the docs
match, and the final report lists any host or step left for Bryan.

## Risks and mitigations

- **Same-stem race between hosts.** The pre-sync under the lock shrinks it to about a
  second. The re-check queues instead of overwriting. If it ever happens, the remote
  wins and a copy lands in `_conflicts/`.
- **Lock contention.** `bob nightly` (03:30) holds `bob_sync.lock` for minutes. Creates
  in that window queue after 60 s, and the timer drains them.
- **`-o lib/...` creates.** The PDF is written before the lock, so vault-sync can commit
  it alone in that gap. The case is rare and explicit; the pre-sync is skipped so the
  session itself never splits it.
- **Parent-note merge conflicts at post-sync.** These follow the existing remote-wins
  policy and are reported, the same exposure `nightly` and `rename` already have.
- **Tests touching the real vault.** The vault-identity check, plus the harness's
  per-command `BOB_VAULT_SYNC_LOCK_FILE` and `XDG_STATE_HOME`, keep tests off `~/bob`.
  Every new test sets `BOB_DIR`.

## Non-goals

- Automating the full `bob ref scan` or running annotation sync off the Mac.
- `bob ref create` JSON output (`bob-cli-5h`). Keep `IntakeOutcome` and the queue slugs
  ready for it.
- The relative-path `bob ref sync` bug (`bob-cli-9a`); the session uses canonical,
  config-derived paths.
- Moving the Mac's annotation scan from cron to a LaunchAgent, and the PDF-tracking
  policy (`bob-cli-1q`).
- Locks for `bob ref sync <pdf>`.
- No bob-mac-capture or bob-plugins changes. The new scan JSON keys are additive, the
  new error code renders through the generic message/hint banner, and the capture
  verdict wording is unchanged.
