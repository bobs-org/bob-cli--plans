---
tier: tale
title: Require sase listen for reference narration
goal:
  Make reference narration launch only sase listen render, migrate direct legacy
  templates, and never fall back to sase-listen.
size: medium
proposed_by: bbugyi200.athena.0zi
create_time: 2026-10-10 17:41:51
status: wip
---

# Require the SASE listen launcher for reference narration

## Outcome and scope

Make `bob ref create <TARGET> -i -L -P sase` invoke `sase listen render` exclusively. An
existing ordinary `sase-listen render {target} -e full -o {audio}` setting must work
through the SASE plugin without running or requiring the standalone executable. There
must be no executable discovery that prefers `sase-listen`, and no fallback to it when
SASE or its listen plugin fails.

This is a medium tale: one coding agent can update the shared narration contract, its
callers' tests, and documentation together. There is one execution boundary and no
useful independently deployable phases. Authoring this tale is large work under the SASE
sizing guidance; implementing it is medium work.

Scope includes the shared listener used by create, its aliases, attach mode, and
background URL ingestion. Preserve configurable narration arguments and the four
existing placeholders. Deliberately narrow the configuration from an arbitrary shell
program to a SASE render invocation: wrappers, alternate executables, and shell programs
are no longer supported. This is necessary to give the requested launcher guarantee;
merely fixing examples or a default would still let an old setting execute the
standalone command.

Do not add CLI options, change capture grammar, install narration tools, run paid
narration, or change the user's vault during implementation. No memory changes are
needed. Keep existing audio-library names and paths containing `sase-listen`; those
identify stored data, not the executable to run.

## Findings and implementation entry points

- `src/native/highlights_ref/listen.rs` resolves `BOB_HIGHLIGHTS_LISTEN_COMMAND` before
  `highlights.listen_command`, trims the setting, validates placeholders, expands a
  shell string, and executes it via `sh -c`. There is currently no default command. The
  user's trace proves that the effective setting in that invocation selected
  `sase-listen`; it does not prove whether that setting came from the environment or a
  config file.
- That file still recommends `sase-listen render` in `NOT_CONFIGURED_HINT` and
  missing-placeholder hints. Its unit tests explicitly accept arbitrary shell syntax.
  `append_listen_doctor_row` checks only the first executable word.
- `src/native/highlights_ref/create.rs` calls `require_validated()` before
  fetching/rendering and uses the same listener for fresh captures and attach mode. Dry
  runs use `would_run_line`. Its long help describes the shell template contract.
- `src/native/highlights_ref/ingest.rs` shares resolution and execution through
  `require_listen_command` / `run_listen_quiet`. Quiet execution must keep JSON stdout
  clean and retain the existing typed error mapping.
- `src/native/config/mod.rs` stores the optional template without interpreting it. Its
  parser tests contain legacy examples. Keep generic config loading separate from
  narration validation so unrelated Bob commands are unaffected.
- `tests/cli/highlights/listen.rs` already covers rendering, PDF URLs, web articles,
  attach, dry runs, interruption, failed/missing audio, and doctor. Its fake listener
  currently works by configuring an arbitrary script path; adapt that fixture to provide
  a fake `sase` executable on an isolated PATH.
- `src/native/env.rs` provides `TestEnvGuard` and `inherit_overrides` for unit tests.
  Never mutate process-global environment in parallel Rust tests.
- `Cargo.lock` already contains `shlex` 2.0.1 transitively. A direct dependency can
  support quoted argument tokenization; do not build a shell interpreter.
- README.md, docs/highlights-create.md, and docs/highlights-ref-sync.md currently
  advertise both executable forms.
- The linked `chezmoi` repository's `home/dot_config/bob/config.yml` already has the
  correct `listen_command: sase listen render {target} -e full -o {audio}`. Its adjacent
  comments still recommend standalone installation and describe shell expansion. The
  live failing configuration was not read or changed.

## Implementation

### 1. Represent and validate a SASE render invocation

Refactor `ListenCommand` to hold argument templates for a fixed launcher rather than an
executable shell string. Retain the existing config key, environment override,
precedence, trimming, and empty/unset semantics: an absent or blank effective setting
still fails early with the canonical configuration snippet. A malformed config file must
still fail as it does today; this change does not introduce an implicit default or
silently ignore overrides.

Accept a single direct invocation beginning with the literal words `sase listen render`.
For compatibility, accept a single direct invocation beginning with `sase-listen render`
and normalize only that launcher to `sase listen render` in memory. Preserve the
remaining flags and arguments, including `{pdf}`, `{title}`, and options such as
`--no-publish`. Do not rewrite the configuration file, URLs, titles, library paths, or
arbitrary substrings. Recognition must use complete tokens, never
`starts_with("sase-listen")`.

Support whitespace and quoted static arguments using a maintained argument tokenizer,
adding the already locked `shlex` version as a direct dependency if used. Keep the
existing rule that placeholders are not manually quoted, and retain required `{audio}`
plus `{target}` or `{pdf}` and unknown-placeholder checks. Expand placeholders within
argument tokens into raw values after tokenization so spaces, quotes, Unicode, newlines,
dollar signs, and shell metacharacters in runtime values remain data in the same
argument.

Reject malformed quoting, NULs, a different executable/subcommand, executable paths,
leading environment assignments, `env`/`command`/shell wrappers, command chains,
pipelines, redirections, and shell substitutions/expansions in the configured template.
Quoted static argument data may contain punctuation; runtime placeholder values must
never be scanned as shell programs. Give a specific diagnostic with the canonical
template and explain that settings now describe SASE render arguments rather than shell
programs. Unsupported legacy wrapper forms should fail before fetch/render, with
instructions to replace them; do not guess at their meaning.

Keep parsing/normalization shared by validation, rendering a dry-run command, doctor,
and execution. Ensure no internal construction path can bypass the launcher invariant.
Add comments describing the limited legacy migration.

### 2. Execute only SASE and preserve the existing lifecycle

In `run_listen_with_output`, use `process::Command::new("sase")` with fixed `listen`,
`render` arguments followed by the expanded configured arguments. Remove `sh -c` from
narration execution. Use shell quoting only to format the human-readable
`listen: run ...` and `listen: would-run ...` display from the actual argv. Neither
display should show the legacy launcher after migration.

Preserve inherited stdin, foreground stdout/stderr, Quiet output forwarding, flush
ordering, signal restoration and interruption exit 130, MP3 validation, scratch
recovery, collision checks, and atomic audio/PDF installation. Keep the existing ingest
error kinds and retry behavior. Forward thread-local test overrides to subprocesses
through the existing environment helper where needed.

If `sase` cannot be launched, report that launcher and suggest checking SASE and its
listen plugin. If the plugin returns an error, retain its streamed diagnostic and the
existing no-vault-write hint. Neither failure may trigger a standalone retry. Do not add
networked plugin installation or render probes.

Update doctor to report the normalized invocation and check the fixed `sase` launcher
rather than extracting an arbitrary executable from the template. Retain non-fatal
warn/ok/none behavior. Describe an `ok` result accurately as launcher
availability/template validity, not proof that the listen plugin or its credentials
work. Recommend `sase listen --help` for plugin diagnosis; doctor does not need a new
potentially slow plugin probe.

### 3. Add regression coverage at the process boundary

Convert test listener fixtures to fake `sase` programs that assert the first two
arguments are `listen render`, log argument boundaries, strip those two arguments, then
use the existing fake MP3/failure behaviors. Scope PATH to each subprocess (or use
`TestEnvGuard` plus explicit child forwarding in unit tests); never invoke an installed
real SASE binary accidentally. Install a fake `sase-listen` trap that records invocation
and fails. Its marker must remain absent on both success and failure paths.

Cover the following contracts with focused unit/integration tests:

- Canonical config and environment override work; the environment still wins. A legacy
  config and a legacy environment override each run the fake `sase` successfully,
  preserving custom flags and never hitting the trap.
- Unset, blank, and invalid settings retain early pre-fetch/pre-render failure. Legacy
  commands with wrappers/chains and non-SASE programs fail early with the new migration
  hint; use fetch and executable markers to prove no work ran.
- No fallback occurs when `sase` is absent but `sase-listen` is available, or when the
  fake SASE reports a missing/failing plugin. Vault files are unchanged.
- Literal input values containing spaces, apostrophes, Unicode, shell syntax, and the
  text `sase-listen` arrive intact as argument data; no naive global replacement
  corrupts them. Unknown/quoted/missing placeholders still fail.
- Dry-run output uses the normalized command and executes neither fake. Doctor uses
  normalized config, checks `sase`, and gives canonical remediation hints.
- Exercise the public `bob ref create` command with `-i -L -P sase` and an offline arXiv
  fixture where available (extend the existing arXiv fake fetch setup as needed). Assert
  the actual `listen render` argv, `-e full`, scratch output path, successful PDF/audio
  binding, and absence of a legacy spawn.
- Retain all existing route/attach/error assertions while changing their fake invocation
  wiring. Cover the shared Quiet runner's clean stdout and typed failure behavior so
  background ingestion cannot regress independently.

Revise shell-specific unit tests to verify the new argument contract instead of
retaining an arbitrary-program bypass just for tests. Keep config parser tests for raw
string/trim behavior, including an explicit legacy-migration fixture at the listener
layer. Do not weaken existing install/collision tests.

### 4. Align help and configuration documentation

Update `src/native/highlights_ref/create.rs` help, all listen configuration and
troubleshooting sections in README.md, docs/highlights-create.md, and
docs/highlights-ref-sync.md, and any affected help assertions. Describe
`sase listen render` as the sole launcher, how direct legacy settings migrate, and why
wrappers/shell programs need conversion. Explain argument passing and the unchanged
placeholder and empty-setting rules. Replace successful command examples and remediation
hints with the canonical form, including historical manual smoke examples that otherwise
instruct readers to run the old command.

Keep factual package identifiers, upstream source provenance, and `sase-listen/library`
paths intact. Remaining executable-form references should only explain legacy
migration/rejection or appear in regression fixtures.

Use `/sase_repo` to reopen `chezmoi` in the implementing agent's workspace and read its
AGENTS.md. Update only the adjacent listen-setting comments in
`home/dot_config/bob/config.yml` to match the narrowed contract; the actual setting is
already correct. Follow that repository's finalizer and mandatory post-commit apply
instructions (`chezmoi update -a --force`). Do not edit live home configuration directly
or make unrelated configuration changes.

## Verification and acceptance

First run focused listener unit and CLI integration tests, then the repository gate from
justfile:

```sh
cargo test --lib native::highlights_ref::listen
cargo test --test cli highlights::listen
just check
```

Include any new ingest/Quiet test in the focused run as appropriate. Use the required
`/sase_monitor` workflow for long-running checks. `just check` is the canonical
formatting, all-target/all-feature Clippy, and full test gate. Do not run real narration
or the user's original URL against the real vault as a test. Review remaining
`sase-listen` matches in tracked source/docs and classify them as explicit legacy
tests/migration text or preserved data/provenance names.

Acceptance requires actual subprocess evidence that both the reported legacy setting and
a canonical setting launch only `sase listen render`; correct normalized dry-run and
doctor output; actionable failure without fallback when SASE is unavailable; unchanged
audio binding/failure atomicity; passing relevant checks; and no implementation claims
based solely on changing documentation.

The final implementation report should call out the deliberate compatibility change for
arbitrary shell templates, the automatic direct-command migration, and the validation
performed. Do not claim the user's installed Bob binary or historical environment has
changed unless deployment was separately performed.
