---
tier: tale
title: Fix mechanical Clippy warnings and gate them on Linux and macOS
goal:
  Preserve behavior while eliminating the selected mechanical lint classes and denying
  them in both canonical lint recipes.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3
create_time: 2026-10-27 11:38:26
status: wip
---

- **PARENT:**
  [202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3.md)

# Fix mechanical Clippy warnings and prevent their return

Implement the `clippy-gate` phase assigned to
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3`. Its accepted design is
`plan:202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md`;
the final decision is `clippy_scope = gate_mechanical`. This tale implements that
decision without reopening it. The underlying task is `bob-cli-v`, which stays open for
the parent epic's land agent. Close only the assigned phase after verification.

The result must preserve Bob's behavior while making the explicitly selected mechanical
Clippy classes fail `just lint` and `just check` on Linux and macOS. Document the
remaining advisory classes in `docs/develop/ci.md`.

## Scope and sizing evidence

The planning tree was clean at `b4a5850d`; the full SHA is recorded on the phase. The
phase was `in_progress` and had no previous implementation notes or unfinished attempt.
The original warning locations have evolved; the gate has not landed: `justfile` defines
only `rustc_gate` and both lint recipes pass that value to Clippy.

On Rust/Clippy 1.95.0, the required baseline command completed successfully in 124.3
seconds:

```sh
cargo clippy --all-targets --all-features --message-format short
```

A supplemental `--message-format json` pass counted primary `(file_name, line_start)`
locations by diagnostic `code`, deduplicating library, library-test, and
integration-test repeats. There are 435 distinct locations across 79 lint classes and
436 class/location pairs. A line can carry more than one lint.
`src/native/highlights_ref` contains 166 distinct locations. The eight classes named in
the epic account for 180 class/location pairs. The full per-class baseline is recorded
on the assigned phase as `PLANNING BASELINE`.

One coding agent can complete this medium tale: compiler fixes handle most of the
selected 286 class/location pairs, six simple propagation/identity sites require manual
equivalents, and gate/documentation changes are confined to two files. There are no new
interfaces or product decisions. Long verification continues through SASE monitors in
the same assignment.

## Exact deny list

Use every lint in this table, including the three currently at zero. These are the only
additional deny-level lints authorized by this tale. The second column counts unique
primary file/line locations per class on the planning baseline.

| Clippy lint                         | Baseline sites |
| ----------------------------------- | -------------: |
| `bind_instead_of_map`               |              1 |
| `bool_assert_comparison`            |              3 |
| `bool_comparison`                   |              3 |
| `chars_next_cmp`                    |              1 |
| `clone_on_copy`                     |              1 |
| `cmp_owned`                         |              3 |
| `collapsible_else_if`               |              0 |
| `collapsible_if`                    |            118 |
| `collapsible_match`                 |              6 |
| `collapsible_str_replace`           |              4 |
| `comparison_to_empty`               |              1 |
| `derivable_impls`                   |              1 |
| `double_comparisons`                |              1 |
| `explicit_auto_deref`               |              1 |
| `extend_with_drain`                 |              3 |
| `filter_map_bool_then`              |              5 |
| `for_kv_map`                        |              1 |
| `items_after_test_module`           |              4 |
| `iter_cloned_collect`               |              3 |
| `len_zero`                          |              3 |
| `let_and_return`                    |              3 |
| `manual_contains`                   |              2 |
| `manual_ignore_case_cmp`            |              1 |
| `manual_is_multiple_of`             |              1 |
| `manual_pattern_char_comparison`    |              3 |
| `manual_range_patterns`             |              1 |
| `manual_rem_euclid`                 |              1 |
| `manual_unwrap_or`                  |              3 |
| `map_identity`                      |              4 |
| `needless_as_bytes`                 |              1 |
| `needless_borrow`                   |             29 |
| `needless_borrows_for_generic_args` |             24 |
| `needless_option_as_deref`          |              2 |
| `needless_return`                   |              8 |
| `op_ref`                            |              2 |
| `print_literal`                     |              2 |
| `question_mark`                     |              9 |
| `redundant_closure`                 |              3 |
| `single_char_add_str`               |              1 |
| `single_element_loop`               |              4 |
| `single_match`                      |              2 |
| `unnecessary_lazy_evaluations`      |              0 |
| `unnecessary_map_or`                |              1 |
| `unnecessary_to_owned`              |              2 |
| `useless_concat`                    |              2 |
| `useless_conversion`                |              0 |
| `useless_format`                    |             11 |
| `while_let_on_iterator`             |              1 |

All current selected sites have machine-applicable suggestions except four
`question_mark` suggestions marked `MaybeIncorrect` and two `map_identity` suggestions
marked `Unspecified`. Inspect and perform those six individually:

- `capture/pomodoro_adjust.rs`, `highlights_ref/sidecar.rs`, `projects/scan.rs`, and
  `highlights_ref/note_snapshot/tests.rs`: each is a
  `let Some(...) = expression else { return None; }` propagation. Preserve the binding
  pattern and evaluate the expression once when replacing it with `?`.
- `capture_language/editor_classify.rs` and `capture_language/tokens.rs`: remove only
  `.map(|(selection, x_len)| (selection, x_len))`.

All those paths are under `src/native/`. Match by code and lint, since line numbers will
move. Preserve comments when removing identity closures; in particular the
timing/overflow explanation in `capture_language/tokens.rs` must remain useful.

The two selected arithmetic classes have bounded, constant-divisor sites:
`highlights_ref/doctor.rs` uses modulo 3 for grouping digits, and
`capture/pomodoro_adjust.rs` normalizes modulo 1440. Keep their output, signed-value
semantics, and casts unchanged. The deny list does not authorize unrelated arithmetic
cleanup.

## Advisory boundary

Do not apply fixes for the following remaining classes or elevate their levels. Document
them as advisory, grouped by why they need separate review. Retain their existing
diagnostic levels and existing narrowly scoped allowances.

- Design, signatures, ownership, and layout: `too_many_arguments` (60),
  `result_large_err` (13), `type_complexity` (7), `large_enum_variant` (5),
  `cloned_ref_to_slice_refs` (4), `ptr_arg` (2), `needless_lifetimes` (2),
  `enum_variant_names` (1), `wrong_self_convention` (1).
- Evaluation, initialization, iteration, allocation identity, arithmetic bounds, and
  panic diagnostics: `expect_fun_call` (6), `unnecessary_sort_by` (3),
  `double_ended_iterator_last` (2), `int_plus_one` (2), `replace_box` (2),
  `needless_range_loop` (2), `unnecessary_unwrap` (2), `missing_const_for_thread_local`
  (1), `explicit_counter_loop` (1), `manual_clamp` (1), `field_reassign_with_default`
  (1).
- Suggestions requiring another intent/readability assessment or lacking a consistently
  mechanical fix: `nonminimal_bool` (5), `unnecessary_get_then_check` (5),
  `format_in_format_args` (3), `match_single_binding` (3), `redundant_locals` (3),
  `doc_lazy_continuation` (2), `if_same_then_else` (2), `match_like_matches_macro` (2),
  `redundant_guards` (2), `assertions_on_constants` (1), `manual_strip` (1),
  `no_effect_replace` (1), `single_range_in_vec_init` (1),
  `suspicious_command_arg_space` (1).

These counts total 150 class/location pairs and can change after other fixes or
concurrent work; they are evidence, not test expectations. For example, the
`expect_fun_call` auto-fix on a `Result` would discard the error value from the panic
message. Machine applicability alone is insufficient to expand this scope.

## Implementation

1. Read the assigned phase, this plan, the accepted epic plan, and `bob-cli-v` through
   audited SASE commands. Read `docs.md` with `/sase_memory_read` before editing the
   developer page. Check the current tree and upstream history for already landed fixes.
   Keep the host-assigned status; do not hand-set it. Refresh the baseline after
   bringing the checkout up to date, recording SHA, toolchain, per-class counts, and
   command exits on the phase. Use `$CARGO_TARGET_DIR` for Cargo builds. Use
   `/sase_repo` before accessing any other repository and audited `sase artifact read`
   for sidecar artifacts. No memory changes, live-vault writes, or plugin deployments
   are authorized or required for this phase.

2. Apply only the selected lints across maintained Rust source and tests, including
   platform-gated code. Prefer `cargo clippy --fix --all-targets --all-features` with
   transient `-A clippy::all` followed by one `-W clippy::<lint>` for each selected
   lint. This limits the fix pass; it is not the persistent lint policy. Use dirty-tree
   allowances only to continue your own reviewed edits, never to absorb another change.
   Inspect the diff by module, starting with `src/native/highlights_ref`, then
   `ref_library`, capture/capture_language, and the other affected modules and tests. Do
   not accept transformations of the advisory classes. Complete the six simple manual
   cases above, format, and repeat selected passes as needed when one fix exposes
   another selected lint. Review condition order, drop/borrow scopes, panic/error text,
   and comments; do not change externally visible output, signatures, or business rules.
   Preserve intentional test cases and assertions.

3. In `justfile`, keep the current `rustc_gate` unchanged. Add one shared value for the
   48 explicit `-D clippy::<lint>` flags and consume it in both `lint` and `check`.
   Avoid `-D warnings`, `-D clippy::all`, duplicated deny lists, and new allow
   attributes that merely hide selected warnings. `check` must still run formatting,
   Clippy, and `cargo test --no-fail-fast`. The existing CI workflow calls `just lint`
   in the Linux lint job and in `test (macos-latest)`; reuse it without adding a new
   workflow or platform-specific lint policy.

4. Add a concise lint-policy section to `docs/develop/ci.md`: list the exact deny
   classes, explain that Linux and macOS use the same gate, list the advisory classes
   and review boundary, and show the commands for lint and full checking. Do not
   describe the entire warning backlog as eliminated. Use developer documentation rather
   than changing CLI help or generated docs. No README, AGENTS.md, toolchain,
   dependency, release-version, changelog, vendored, or generated-file changes belong to
   this work.

## Verification and integration

1. Run the normal all-target/all-feature Clippy pass and the selected deny gate.
   Recompute deduplicated diagnostics, proving zero selected warnings/errors and
   recording the advisory remainder. Run `just --dry-run lint` and
   `just --dry-run check` to confirm both recipes consume exactly the same list and
   retain the existing compiler flags. Existing Rust tests are the regression coverage
   for these mechanical edits; add a focused test only if a non-obvious equivalence
   needs one, not one test per compiler rewrite.

2. Check the macOS conditional branches before landing. With the installed
   `aarch64-apple-darwin` target, use disposable stub `CC_aarch64_apple_darwin` and
   `AR_aarch64_apple_darwin` scripts plus `CRATE_CC_NO_DEFAULTS=1`, as the parent plan
   requires, to run:

   ```sh
   cargo clippy --target aarch64-apple-darwin --all-targets --all-features -- \
     -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces \
     <the same 48 explicit Clippy deny flags from justfile>
   ```

   Read the flags from the shared justfile value rather than maintaining a second list.
   Apply/review selected fixes on macOS-only code too. Keep disposable stubs out of the
   source diff. This proves type/lint coverage, not runtime correctness.

3. Run the full `just check` on the final source tree, preferably through
   `sase tool run check`, and `just docs-check`. Honor `/sase_monitor` for long commands
   and any tool escalation; join an escalated ToolRun instead of restarting it. After a
   monitor, continue the assigned work from its frozen result. Do not run
   `just check-full`.

4. Before landing, run `scripts/dev/macos-ci-check` through `/sase_monitor` with profile
   `verify`, a timeout of at least 95 minutes, and a continuation that inspects every
   job and required macOS step, repairs phase-caused failures, and completes this phase.
   Wait for the monitor-start command itself to exit. Record snapshot SHA, run URL,
   workflow conclusion, macOS lint/tests/presenter test verdicts, and verified
   scratch-branch cleanup. Exit 0 passes; exit 1 is a real failure; exit 2 is
   unverified. Do not claim stub cross-checking supplies the missing runtime verdict. No
   interactive Mac session is necessary.

5. Concurrent agents edit `highlights_ref` and `ref_library`. Rebase immediately before
   the commit/finalization boundary, preserving others' work. When a mechanical change
   conflicts, regenerate that change against the rebased file with the scoped Clippy
   pass instead of manually combining stale rewrites. Rerun the necessary checks if code
   or the tested tree changes. Keep module groups reviewable in the diff and, where the
   authorized commit workflow allows, in separate commits. Never use raw `git commit`;
   follow SASE's authorized commit/finalization workflow. Verification must cover the
   actual tree committed, and the parent lander must inspect macOS CI on the landed SHA
   as well.

6. If a check fails, distinguish introduced errors from existing defects. A failure that
   reproduces identically on the clean base does not keep this phase open. Record its
   commands, base SHA, failure identity, and existing task ID if known as
   `PROPOSED FOLLOW-UP:` on this phase and report the limitation honestly. Do not
   quietly label an unrun CI check a baseline failure. Fix regressions from this tale
   before declaring completion. Create no task beads or unrelated fixes.

## Completion contract

Record the before/after inventory, selected policy, changed module groups, tested
tree/commit, Linux/docs results, cross-target result, and macOS CI evidence on
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3`. Note that the parent lander owns the
resolution of `bob-cli-v` under the accepted narrower scope. Append any out-of-scope
discovered work using:

```sh
sase bead note bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3 \
  'PROPOSED FOLLOW-UP: <one-line summary — evidence and detail>'
```

Immediately before closing, run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3
```

Resolve any reported symbol or re-key its justfile line to a still-open parent or later
phase; the planning-time query found none. Then close only this phase:

```sh
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.3 \
  --note '<exact implementation and verification evidence>'
```

Do not close `bob-cli-v`, the parent epic, or any ancestor plan bead. Do not start the
parent lander's backlog campaign or deployment work. Before a normal final response, use
`/sase_final` as the last action; a successful SASE monitor handoff is the exception
when verification needs to continue.
