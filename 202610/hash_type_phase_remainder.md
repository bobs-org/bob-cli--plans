---
tier: tale
size: medium
title:
  Finish phase bob-cli-ep.1 — attached `#<type>` forms, refusal copy, stale docs, and
  the capture-parse budget
goal:
  Attached `#<type>` works in leading position and with spaced literals, refusals carry
  the epic copy and hints, docs stop describing the deleted `#tag`/named Ensure Next
  behavior, the PT3 budget is shown met by a paired A/B, and bob-cli-ep.1 closes.
proposed_by: bbugyi200.apollo.bob-cli-ep.1
proposed_with: claude/opus
bead: bob-cli-ep.1
create_time: 2026-10-31 08:16:18
status: wip
---

- **BEAD:**
  [bob-cli-ep.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ep/bob-cli-ep.1.md)

# Plan: finish phase `bob-cli-ep.1` (`#<type>` in bob-cli)

## Context

Phase `bob-cli-ep.1` belongs to epic `bob-cli-ep`, whose approved plan is
`plan:202610/capture_hash_type_marker.md`. Read that plan's §1–2 and the "Phase `cli`"
list first. They are the normative contract, and the epic DECISIONS shown by
`sase bead read bob-cli-ep.1` are final:

- `bare_hash_menu = letter`;
- `retire_t = prose`;
- `memory_record = no`. No memory note may be edited.

Commit `4a95e5b7` (`feat(capture)!: replace t: type markers with #<type>`) landed the
migration. The phase was left open for one reason: an apollo run measured
`capture-parse 'Fix the parser #f'` at p95 53.6 ms, against a 49.7 ms limit. The limit
is PT3's "+5 ms p95 over the 44.7 ms baseline".

Re-checking the phase found two things:

1. **The latency miss is host-load noise, not a regression.**
   - Setup: on 2026-10-31 on apollo, two debug builds were compared against the live
     vault in interleaved rounds.
     - Base: `00ca6d0c`, the parent of `4a95e5b7`, running `t:f`.
     - HEAD: running `#f`.
   - Results:

     | Run                   | Request            | Base p95 (`t:f`) | HEAD p95 (`#f`) |
     | --------------------- | ------------------ | ---------------- | --------------- |
     | 200 rounds            | `capture-parse`    | 41.5 ms          | 40.3 ms         |
     | 60 rounds, under load | `capture-parse`    | 52.4 ms          | 52.0 ms         |
     | 60 rounds, under load | `capture-complete` | 46.8 ms          | 44.4 ms         |
     | 60 rounds, under load | zsh protocol       | 62.5 ms          | 66.8 ms         |

     The 200-round run's p50 was 32.3 ms (base) and 32.4 ms (HEAD).

   - The load average was 5–9 from other agents. Absolute p95 moves by about 10 ms with
     that load, so a single-binary 20-sample run cannot judge a +5 ms budget. A paired
     delta can.

2. **A conformance sweep against the epic contract found real gaps.** These were
   reproduced with the HEAD debug binary and `bob capture --dry-run` / `capture-parse` /
   `capture-complete` against the live vault. They are the real remaining work, listed
   under Work below.

One coding agent can finish all of it. Every change sits in the type-marker lexer, its
completion subject, the refusal message, docs, and tests.

## Ground rules

- **Allowed edits.** Do not edit `sase/memory/**`, `docs/develop/decisions.md`
  (generated from the decisions web), `Cargo.toml`'s version, or `CHANGELOG.md`. Do not
  touch `tests/fixtures/pomodoro_themes/vectors.json`.
- **Docs rules.** Before editing `docs/` or CLI help, run
  `sase memory read docs.md -r "<why>"`.
- **Thin client.** The wire shapes stay additive and `schema_version` stays 1.
- **Commit subject.** Use something like
  `fix(capture): finish attached #<type> forms, refusal hints, and docs`.
  - This is a fix to an already-announced break, so no `!` and no `BREAKING CHANGE:`
    footer.
- **Bead.** Do not create beads. Record anything out of scope as
  `sase bead note bob-cli-ep.1 'PROPOSED FOLLOW-UP: …'`.
- **Mac.** The Mac phase (`bob-cli-ep.2`) is separate. Do not touch bob-mac-capture.

## Work

### 1. Lexer: attached forms at every marker position

Edit `src/native/capture_language/type_marker.rs`. Keep the shared predicates; the
classifier in `completion.rs` and `editor_classify.rs` must agree with the lexer.

**a. Leading attached route tokens.**

- Repro: `@sase^zzqq-new#fix Fix the parser` fails in `capture-parse`,
  `capture --dry-run`, and completion with the retired `invalid_project_note_marker`
  ("needs the project-note `+`").
  - With the cursor after `#f`, completion returns `context: pomodoro_name` with
    un-hashed replacements.
  - The trailing form `Fix the parser @sase^parser#fix` works.
- Cause: `attached_route_candidate` sets `edge_start` to the `#`. The leading-edge check
  then walks from 0 over the route prefix, which is not in `allowed`, so the candidate
  is dropped.
- Fix: judge position (§1.2) using the whole route token. Either add
  `token_start..token_end` to `allowed`, or use it for the edge test. Keep the existing
  replacement-spacing behaviour: an attached suffix never adds a space.
- Also check the leading `@sase+parser#fix &notes:spec`. A `+` form with body text must
  stay a task section.

**b. Attached literals containing whitespace.**

- Repro, in parse, completion, and execution:
  - `Fix the parser @sase^parser#"DEEP WORK"` and its curly variant `#“DEEP WORK”` fail
    with `invalid_project_note_marker`.
  - `@sase+parser#"DEEP WORK"` fails with `invalid_sub_bullet_section`.
  - With the cursor at the end, completion on `…@sase^parser#"DEEP W` gives
    `context: pomodoro_name`, and on `@sase+parser#"DEEP W` it gives `task_section`.
  - Literals without spaces (`#"Q&A"`) and unterminated literals (`@sase^parser#"DEEP`)
    already work.
- Cause: the attached pass iterates `whitespace_tokens(body)`, which splits the literal
  at the space.
- Fix: when the suffix opens a literal (`"` or `“`), extend the token through the
  closing quote with `lex_exact_name_after_hash`. This is the same rule as the
  free-standing literal and `merge_leading_literal_token`.
  - Use the extended end for the candidate, for masking, and for the edge check.
  - An unterminated attached literal runs to the end of the line and is Incomplete, so
    it needs `task_type`.
- Ensure `marker_only_task_toggle` sees the whole literal.

**c. Completion `subject` for an empty attached `@r^id#`.**

- Repro: `Fix the parser @sase^zzqq-new#` (cursor at the end) reports
  `subject: "existing"`. With `#f` it correctly reports `"new"`.
- Cause: in `src/native/capture_complete/task_types.rs`, `whole_item_link` treats any
  `EditorMode::Incomplete` item with a `block_id` as an existing link.
- Fix: decide from the item's spans. A `task_block_id` span means new; a
  `task_toggle_block_id` span or a whole-item link (`pomodoro_link` route/id) means
  existing.
  - Marker-only `@sase+parser#` must stay `"existing"`. It is already tested in
    `tests/cli/capture/complete_task_types.rs`.

### 2. Execution: refusal copy and context hints (epic §1.4)

Edit `type_not_configured_error` in `src/native/capture/task_theme.rs`. It has two
callers (new task, about line 174; existing task, about line 403).

**Copy fixes:**

- **Did-you-mean.** Today the output reads
  ``did you mean `#BUILD`?; add `BUILDS` under …``. Change it to:
  - at most two names from `catalog.similar_to`, joined by `or`;
  - each name in marker spelling, using `replacement_for(name, key)`.
    - Move that helper from `capture_complete/task_types.rs` into
      `capture_language/type_marker.rs` and share it.
    - Examples: `#build`, `#"2FA"`.
  - The clause is omitted when nothing is similar. Keep the existing similarity rule:
    `#fxi` → FIX is not similar (3 characters, a transposition), so it gets no
    suggestion.
- **Wording, with a vocabulary.** Use the epic copy exactly. `<config>` is the path Bob
  already prints. The expected strings are below; the second assumes both CHORE and
  CHORUS are configured, CHORE used more recently:

  ```text
  `#builds` is not one of your task types; did you mean `#build`? Add BUILDS under `types:` in <config> to use it, or write `\#builds` to keep it as a tag
  `#chores` is not one of your task types; did you mean `#chore` or `#chorus`? Add CHORES under `types:` in <config> to use it, or write `\#chores` to keep it as a tag
  `#fxi` is not one of your task types; add FXI under `types:` in <config> to use it, or write `\#fxi` to keep it as a tag
  ```

- **Wording, without a vocabulary.** Keep the current copy:

  ```text
  `#snack` is not a task type yet: no Pomodoro has used SNACK and no `types:` vocabulary is configured; add `types:` to <config>, or write `\#snack` to keep it as a tag
  ```

- **Authored spelling.** The marker and its escape quote the authored token:
  `#"DEEP WURK"` and `\#"DEEP WURK"`, never `#DEEP WURK`.
  - Pass the authored token text, or enough of the marker, into the refusal.
  - The NAME in "Add NAME under" stays `unmatched_type_display(name)`.

**Context hints.** Append at most one hint, in parentheses, at the end. They apply to
both the vocabulary and no-vocabulary copy.

The three hints:

- **On an attached `@r^id#NAME`.** Drop the escape clause and append the project-note
  hint.
- **On an attached marker-only `@r+id#NAME`.** Drop the escape clause and append the
  Pomodoro-link hint.
- **On a free-standing marker** in an item whose route token is a bare `@route` (no `:`,
  `^`, `+`, or `#` part), as in `#fxi Fix it @sase`: keep the escape clause and append
  the heading hint.

Expected strings, in that order, for `Fix it @sase^parser#fxi`, `@sase+parser#fxi`, and
`#fxi Fix it @sase`:

```text
`#fxi` is not one of your task types; add FXI under `types:` in <config> to use it (for a project note whose ` :id` tasks go under the FXI Pomodoro, write `@sase^parser+#fxi`)
`#fxi` is not one of your task types; add FXI under `types:` in <config> to use it (to link the task under the FXI Pomodoro, write `^sase:parser#fxi`)
`#fxi` is not one of your task types; add FXI under `types:` in <config> to use it, or write `\#fxi` to keep it as a tag (for a bullet under a heading, write `@sase#fxi`)
```

- **Spelling inside hints.** `@r`, `id`, and `name` use the authored route, block id,
  and the authored text after `#`.
- **Carrying the attachment.** The refusal must know how the marker attached. Prefer
  carrying it from the lexer: for example, add an attachment field to `TypeMarker`
  (`Free` / `ProjectTask { route, block_id }` / `ExistingTask { route, block_id }`). Do
  not re-scan raw text in `plan.rs`.

### 3. Docs

Update each place below, then grep for the same phrases elsewhere.

- **`docs/task-types.md` PT12**, the "Warning code" bullet. It still says "Warnings
  never block anything; there is no strict mode".
  - Add the epic's one stated exception: the capture `#<type>` marker refuses
    unconfigured names with `type_not_configured`.
  - Link to `### Typed captures (#<type>)`.
- **`docs/capture.md`:**
  - **Rows ~277–280** (`Fix it @sase #now`, `Fix it #now @sase`, `@sase:deep-fix #now`,
    `Fix it @sase #n`). They still describe the deleted trailing-`#tag` / legacy
    bullet-marker errors. Rewrite them as `type_not_configured` refusal rows: unknown
    type unless configured, with the heading hint where a bare `@route` is present.
    - Add rows for `@sase^parser#fxi` (project-note hint) and `@sase+parser#fxi`
      (`^r:id#name` hint).
  - **The paragraph after the table (~282)**: "with no body text it names a Pomodoro for
    the task toggle". Say instead that with no body or children the suffix is the task
    type (`#<type>`), and with body text it is the ALL-CAPS section.
  - **`### #now is retired` (~1065)**: "A trailing `#now` is now the ordinary
    trailing-tag error". Say instead that `#now` is an ordinary unknown-type refusal
    unless `NOW` is configured, and `\#now` keeps it as a tag.
  - **"Typing tasks with `#<type>`" (~487–530)**:
    - state the refusal copy shape (did-you-mean, the three hints);
    - say that attached forms work in leading and trailing positions;
    - say that attached literals may contain spaces.
- **`docs/reference/capture-syntax.md`:**
  - **`{#plus-child-section}` (~1154)**: "With no other text, `#` names a Pomodoro
    instead (Ensure Next / move)." Change it to: with no other text or children, the
    suffix is a task type ([`#<type>`](#hash-type)).
  - **`{#plus-ensure-next}` (~1171)**:
    - Drop "`#pomodoro` moves the subtree to that named open Pomodoro, creating a named
      future Pomodoro when missing".
    - Say that `@route+block-id#<type>` retypes the task first so the link lands by the
      new type.
    - Say that `^route:block-id#NAME` links under a named Pomodoro.
  - **The Overloaded-markers `#` rows (~106–109)**: add rows for free-standing `#<type>`
    at an item edge, `@route^id#type`, and `@route+id#type` with no body or children.
  - **The Retired rows (~150–158)**: turn the "`#now` / a free `#tag` next to `@route`"
    row into a refusal row; `t:` is ordinary prose.
- **`docs/completion.md` Performance**: rewrite after step 5.
- **Final sweep.** Run
  `git grep -n -i -E "trailing.tag error|retired bullet marker|names a Pomodoro for the task toggle|Ensure Next / move|bullet section markers" -- docs src`
  outside `docs/develop/decisions.md` and `docs/reference/mac-capture/`. It must come
  back empty.
- **Regenerate.** If clap help text changes, run `just docs-gen`.

### 4. Tests

Write these first; each must fail on HEAD before your fix.

- **`src/native/capture_language/tests/type_marker_editor.rs`.** Extend
  `masked_items_match_their_baseline_across_grammar_families` (the masking-parity test
  the epic calls "the property test") with attached forms:

  ```rust
  check_masking_parity("Fix the parser @sase^parser", "Fix the parser @sase^parser#fix", 27, "#fix");
  check_masking_parity("Fix the parser @sase^parser", "Fix the parser @sase^parser#\"DEEP WORK\"", 27, "#\"DEEP WORK\"");
  check_masking_parity("@sase^parser Fix the parser", "@sase^parser#fix Fix the parser", 12, "#fix");
  check_masking_parity("@sase+parser", "@sase+parser#fix", 12, "#fix");
  check_masking_parity("@sase+parser", "@sase+parser#\"DEEP WORK\"", 12, "#\"DEEP WORK\"");
  check_masking_parity("@sase+parser &notes:spec", "@sase+parser#fix &notes:spec", 12, "#fix");
  ```

  Adjust the offsets if the helper needs it. The point is a span-for-span baseline
  equality plus exactly one `task_type` span.

- **Lexer and editor unit tests**:
  - the spans touch but never overlap for each new form, including the curly literal;
  - incomplete `@sase^parser#"DEEP` / `#"DEEP W` need `task_type`;
  - a mid-line `Fix @sase^id#fix the parser` keeps today's behaviour.
- **`tests/fixtures/capture_type_marker/vectors.json`**: add
  `{"input":"Fix the parser @sase^parser#fix","body":"Fix the parser","type":"fix"}`,
  `{"input":"@sase^parser#fix Fix the parser","body":"Fix the parser","type":"fix"}`,
  and the `#"DEEP WORK"` attached literal.
  - Use the vector's `type` field as the existing test expects: decoded name.
- **`tests/cli/capture/capture_type_marker.rs`** (temp vault and config with `types:`
  including `DEEP WORK`):
  - leading `@sase^parser#fix Fix the parser` writes `[type:: FIX]` on `^parser`;
  - `@sase^parser#"DEEP WORK"` writes `[type:: DEEP WORK]`;
  - each refusal string from step 2 is matched exactly:
    - did-you-mean with one name, two names, and none;
    - a literal refusal;
    - the `^` hint, the `+` hint, and the heading hint;
    - the no-vocabulary copy with an attached hint.
  - The snapshot stays unchanged after each refusal, in both dry-run and real runs.
- **`tests/cli/capture/type_marker_existing.rs`**: `@sase+parser#"DEEP WORK"` retypes
  `^parser` (Ensure Next placement by the new type).
- **`tests/cli/capture/complete_task_types.rs`**:
  - `Fix the parser @sase^new-id#` returns `context: task_type`, `subject: "new"`, and
    the full configured menu;
  - `@sase^new-id#f Fix the parser` (cursor after `#f`) returns `task_type` with `#fix`
    replacements and no added space;
  - `…@sase^parser#"DEEP W` and `@sase+parser#"DEEP W` return `task_type`.
- **Existing suites stay green**, including `tests/cli/task_move.rs` and the `@r:id#` /
  `=#` Pomodoro-name completion tests.

### 5. Re-measure the PT3 budget as a paired A/B

The budget is "warm `capture-complete` and `capture-parse` stay within +5 ms p95 of
baseline on apollo against the live vault". Judge it on the paired delta, after steps
1–4 are in.

**Build both binaries.** Use a scratch worktree under the gitignored `target/` and a
separate target dir:

```bash
git worktree add --detach target/hash-type-perf/base-src 00ca6d0c
(cd target/hash-type-perf/base-src && CARGO_TARGET_DIR="$PWD/../base-target" cargo build --bin bob)
cargo build --bin bob
cp target/hash-type-perf/base-target/debug/bob target/hash-type-perf/base-bob
cp "${CARGO_TARGET_DIR:-target}/debug/bob" target/hash-type-perf/head-bob
```

**Run the paired benchmark.** Run from a neutral cwd with `BOB_DIR=~/bob`. The commands
are read-only (`__complete` never writes). Run at least 200 interleaved rounds, with
this script saved outside the repo (for example `/tmp/hash-type-bench/bench.py`) and run
with `python3 -I`:

```python
"""usage: bench.py ROUNDS LABEL=BIN::ARG1::ARG2... — interleaved, one warmup per case."""
import subprocess, sys, time
def pct(xs, q):
    xs = sorted(xs); k = max(1, -(-len(xs) * q // 100)); return xs[int(k) - 1]
rounds = int(sys.argv[1])
cases = [(s.split("=", 1)[0], s.split("=", 1)[1].split("::")) for s in sys.argv[2:]]
run = lambda argv: subprocess.run(argv, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
for _, argv in cases: run(argv)
samples = {label: [] for label, _ in cases}
for r in range(rounds):
    for label, argv in cases[r % len(cases):] + cases[:r % len(cases)]:
        t0 = time.perf_counter(); run(argv); samples[label].append((time.perf_counter() - t0) * 1000)
for label, xs in samples.items():
    print(f"{label:22s} n={len(xs)} p50={pct(xs,50):.1f} p95={pct(xs,95):.1f}")
```

**The cases:**

| Request          | Base                                                                        | HEAD                                                                       |
| ---------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| capture-parse    | `capture-parse --format json -- 'Fix the parser t:f'`                       | `capture-parse --format json -- 'Fix the parser #f'`                       |
| capture-complete | `capture-complete --cursor 18 --format json -- 'Fix the parser t:f'`        | `capture-complete --cursor 17 --format json -- 'Fix the parser #f'`        |
| zsh protocol     | `__complete zsh --protocol 1 --suffix '' -- bob capture Fix the parser t:f` | `__complete zsh --protocol 1 --suffix '' -- bob capture Fix the parser #f` |

Pass the empty `--suffix` value as an empty `::` field.

**Pass or fail.** Each HEAD p95 must be at most its base p95 + 5 ms.

- If a delta fails, re-run once.
- If it fails twice:
  1. Profile it. The catalog load in `capture_parse::run` →
     `ThemeCatalog::load_for_theme_keys` is about 12 ms of the request.
  2. Fix any regression this plan introduced before closing.
- Afterwards, remove the scratch worktree:
  `git worktree remove --force target/hash-type-perf/base-src && rm -rf target/hash-type-perf`.

**Update `docs/completion.md` § Performance:**

- Replace the three task-type rows with the HEAD p50/p95 from the paired run.
- Replace the "`capture-parse` is 3.9 ms above its 49.7 ms limit" sentences with the
  paired method and its deltas:
  - the date;
  - the rounds;
  - base `00ca6d0c` with `t:f`;
  - each delta;
  - why the budget is judged on the paired delta under shared-host load.

### 6. Verify and close

1. Run `just check`, `just docs-check`, and `just all`. All must pass.
   - A failure that also reproduces on a clean checkout of `e9c62aeb` or later master is
     recorded as `PROPOSED FOLLOW-UP:` and does not block.
2. Re-run the sweep below with the new debug binary. Use `--dry-run` against a temp
   vault, or rely on the CLI tests above, and confirm each outcome:
   - `@sase^parser#fix Fix the parser` → new typed task.
   - `@sase^parser#"DEEP WORK"` in leading and trailing positions → typed.
   - `@sase+parser#"DEEP WORK"` → retype plus Ensure Next.
   - `Fix it #builds` → did-you-mean `#build`.
   - `Fix it @sase^parser#fxi` → project-note hint.
   - `#fxi Fix it @sase` → heading hint.
   - These are unchanged:
     - `jot idea #` → Pomodoro note, no completion;
     - `Ask about #fix today` → prose;
     - `Review PR #1234` → captures;
     - `Fix it t:fix` → body keeps `t:fix`;
     - `@sase:deep-fix#bugsxyz Fix it` → warns, does not refuse;
     - `@sase+parser#fix!` → the toggle error copy.
3. `git grep -n 't:'` over `src/native/capture*`, `docs/`, and `tests/` finds only
   intentional prose cases and unrelated text (`[t:: 30m]`, relation targets like
   `~t:tgt`).
4. Run `sase bead epic-symbols bob-cli-ep.1`. Resolve any entries; none are expected.
5. Run
   `sase bead close bob-cli-ep.1 --note "<what you verified, including the paired p95 deltas>"`.
   - Do NOT close epic `bob-cli-ep` or any ancestor.

## Non-goals

- No change to Pomodoro-name `#NAME` resolution, warnings, or completion.
- No change to the similarity rule. `#fxi` → FIX stays unsuggested.
- No Mac (`bob-cli-ep.2`) work, and no memory or decision-record edits.
- No committed benchmark script or new `just` recipe.
