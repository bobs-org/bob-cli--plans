---
tier: epic
title: Split embedded assets into a release-managed workspace crate
goal: Complete bob-cli-da.4 with byte-identical embedded assets in a separate crate,
  both archives verified below their size budgets, and the daily release train supporting
  the workspace without changing Bob's installed behavior.
phases:
- id: assets-and-packages
  title: Move embedded payloads and verify both Cargo packages
  size: medium
  depends_on: []
  description: 'assets-and-packages: create the bob-obsidian-assets workspace crate,
    move the existing font and vendor payloads without changing their bytes, preserve
    installed and source tree consumers, and extend package-check to verify both archives
    with their own license and size budgets; add unique release tags with the new
    member. Follow the assets-and-packages section below.'
- id: release-integration
  title: Integrate the assets crate with the daily release train
  size: medium
  depends_on:
  - assets-and-packages
  description: 'release-integration: verify independent assets versioning and unique
    tags under the existing release-plz train, cover multi-package release output
    and preview behavior, update release documentation, and supply final package measurements.
    Follow the release-integration section below; leave closure of bob-cli-da.4 to
    this child epic''s land agent.'
proposed_by: bbugyi200.athena.bob-cli-da.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-da.4
create_time: 2026-10-27 16:43:42
status: done
bead_id: bob-cli-da.4.1
---

- **PROMPT:** [prompts/202610/embedded_assets_split.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/embedded_assets_split.md)
- **PARENT:** [202610/backlog_batch_b2.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b2.md)
- **BEAD:** [bob-cli-da.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-da/bob-cli-da.4.1.md)

# Split embedded assets out of bob-obsidian

## Objective and authority

Complete assigned phase `bob-cli-da.4` from `plan:202610/backlog_batch_b2.md`. Its final
decision is `assets_split = split_now`; there is no deferred branch or new feature
decision. The task motivating the phase is `bob-cli-ai`.

This child epic covers the assets split only. It does not inherit the parent epic's
requirement to plan the next backlog batch. Leave that to `bob-cli-da`'s land agent. Do
not edit memory, bump the existing root package version, edit CHANGELOG.md, publish
crates, change rehearsal mode, or change the train schedule.

The latest user instruction restricts completion to `bob-cli-da.4`. Although the older
parent plan says to close `bob-cli-ai`, leave that task and `bob-cli-da` open and
provide evidence for the parent lander. Child phase workers close only their own
assigned child phase; the child epic's land agent completes `bob-cli-da.4` after
integration verification.

## Investigation and sizing

At planning time the checkout is clean at `f00854be`. The assigned phase is in_progress,
with no prior implementation notes. No assets crate or Cargo workspace exists, and the
root manifest still packages all embedded payloads. The split has therefore not already
landed. Do not hand-edit bead status or reopen the already active phase.

The audited `bob-cli-ai` note reports the latest master CI package measurement:
8,216,260 bytes, 7.84 MiB, leaving about 0.16 MiB before the warning. This is historical
evidence, not a new measurement of this checkout. Capture a fresh baseline before moving
files. The existing size-only command is
`python3 tools/release/package_check.py --no-verify`.

Current uncompressed payload groups are approximately 5.8 MiB in `assets/print_fonts`,
6.0 MiB in `scripts/web_clip/vendor`, and 0.75 MiB in `scripts/web_clip/fonts`. The root
manifest and `tools/release/package_check.py` currently assume one package. Rust
constants in `src/print_fonts.rs` are shared with `src/scripts.rs`; three Python
consumers and three docs generators also locate the source assets directly.
`release-plz.toml` currently gives every package the tag pattern `v{{ version }}`, which
would collide after adding a second crate. These are two bounded implementation units:
an atomic asset and package migration, then release integration and its documentation.

Required context already consulted: `sase_beads.md`, `sase_sizes.md`,
`sase_artifacts.md`, `docs.md`, and `decisions:bob-cli-releases-ride-a-daily-train`.
Workers should read the relevant memory through `sase memory read` when needed, never
directly.

## Fixed design

1. Add a nonvirtual Cargo workspace containing the existing root package and
   `crates/bob-obsidian-assets`. Keep ordinary root commands working, with the root as
   the default member and explicit workspace coverage in verification. Match the
   existing edition and Rust 1.95 toolchain; use resolver 3.
2. Initialize the new library crate at `0.1.0`, versioned independently. Add a normal
   root dependency with both `path` and `version`. Future versions and dependency bumps
   belong to release-plz. Do not change the root `0.4.0` version manually. The assets
   crate has no runtime dependencies or build-time downloads.
3. Move the complete three payload directories into the new crate, preserving their
   relative layout: `assets/print_fonts/`, `scripts/web_clip/fonts/`, and
   `scripts/web_clip/vendor/`. Include every existing license, notice, and provenance
   README alongside its payload. Do not upgrade fonts or vendor bundles. Leave Python
   scripts, reader CSS, snapshot.js, and small native assets in the root package. Move
   files, rather than keeping duplicate source payloads or symlinks that Cargo would
   flatten into the main archive.
4. Export a small typed API of `&'static [u8]` constants backed by `include_bytes!`
   inside the assets crate. Keep `src/print_fonts.rs` as the internal facade and retain
   its shared-byte tests. `src/scripts.rs` references those constants. The existing
   installed paths, bytes, executable flags, and the four binary names stay the same.
   Update source_path metadata to the new repository locations consistently; a one-time
   embedded-cache refresh is acceptable.
5. Check each compressed archive independently with the existing strict-above 8 MiB
   warning and 9 MiB failure thresholds. Aim for both below 8 MiB with meaningful
   headroom. Keep the 10 MiB cap display and contributor diagnostics. Full package
   verification must work before the new crate exists on crates.io.
6. Both crates remain managed by the same scheduled release-plz workflow. Keep the root
   tag `v{{ version }}` and give assets `bob-obsidian-assets-v{{ version }}`. Disable
   GitHub Release creation for the assets crate while retaining its version tag and
   release processing. This preserves the existing root Release and binary distribution
   contract and avoids counting an assets-only GitHub Release in the daily guard. Both
   packages inherit the current workspace rehearsal setting. Do not set
   `publish = false`, `release = false`, or `publish_no_verify = true` to bypass
   required work.

Cargo's stable workspace packaging supports interdependent packages; select both
together rather than invoking root packaging against an unpublished dependency. See the
[Cargo package reference](https://doc.rust-lang.org/cargo/commands/cargo-package.html)
and
[Rust's workspace publishing announcement](https://blog.rust-lang.org/2025/09/18/Rust-1.90.0/).
Package-specific tags and GitHub Release settings are documented in the
[release-plz configuration reference](https://release-plz.dev/docs/config). Use the
project's pinned release-plz **0.3.173** to verify configuration compatibility; the
planning host's global binary is **0.4.1** and is not evidence for the pinned version.
Do not silently upgrade the pin.

## Phase assets-and-packages

Implement the asset move and package gate in one change so a completed phase has
buildable, verifiable archives.

- Capture the current package size and an inventory/hash comparison for the payloads.
  Create the new crate's manifest, minimal library, README, and MIT license. Its include
  list must ship all payloads and notices and only its necessary source/metadata. Remove
  moved payload include entries from the root manifest. Update Cargo.lock through Cargo
  with the smallest necessary change.
- Move the directories and wire Rust constants through the new crate. Preserve sharing
  of the eleven web-reader print faces with note-snapshot fonts; all thirteen print
  faces remain available to their existing users. Keep the existing checks that exclude
  variable WOFF2 and omit display/emoji faces from the web-reader install set.
- Update `scripts/web_clip/web_clip_adapter.py`, `web_clip_render.py`, and
  `diagram_render.py` so installed assets remain preferred and direct source execution
  resolves moved vendor files, print fonts, DejaVu, and notices. Cover
  `_reader_font_dirs`, font lookup, vendor constants, and the self-test's direct path
  assumptions. A small shared path helper is acceptable; if added, include and embed it
  as a runtime script. Preserve unsafe-font-name rejection. Do not change capture
  extraction semantics: parent sibling phase `bob-cli-da.2` owns the Mermaid class fix
  in web_clip_adapter.py. Re-read that file before edits and preserve any landed sibling
  changes.
- Update the source paths in `tools/docs/bob_docs/assets.py`, `fonts.py`, and
  `media.py`, their relevant tests, and moved asset provenance links where necessary.
  Preserve generated documentation asset bytes; relocation should not cause font or
  image regeneration churn.
- Refactor `tools/release/package_check.py` to enumerate publishable workspace members
  from Cargo metadata and run `cargo package --workspace --locked --allow-dirty`,
  preserving the optional local-only `--no-verify` mode. Inspect/list each package
  separately, associate it with its own archive, and fail on missing expected
  packages/archives. The root now requires MIT and Moment notices; assets requires MIT
  and every moved OFL, DejaVu, Defuddle, and Mermaid notice. Preserve forbidden path
  checks for each archive. Ensure the root archive does not also contain the moved
  payload tree and each root include_bytes input is still available.
- Extend `test_package_check.py` with meaningful cases for both packages, missing
  notices, missing archives, a size failure in either archive, unchanged threshold
  boundaries, and full verification being the default. Use stable Cargo output, not the
  unstable JSON package-list format.
- Update `justfile` Cargo check/test/lint/format recipes and CI test invocations as
  needed to cover both members, preserving root install/binary behavior. Make
  package-list show both packages without mixing their inventories.
- Add the minimal explicit package overrides in `release-plz.toml` for the fixed
  root/asset tag patterns and assets GitHub Release policy in this phase, alongside the
  new workspace member. The intermediate completed phase must not inherit a colliding
  tag pattern while waiting for release integration.

Verify byte equality against the pre-move inventory, shared-font tests, embedded asset
installation and source-independent use, and source-tree adapter self-tests. Exercise
both source and extracted-installed paths so a test cannot accidentally pass by reading
files left in the checkout. Run `just check-release-tools`,
`just check-web-clip-adapter`, relevant docs font/asset tests, and `just package-check`
with full build verification. Record any browser-dependent skips explicitly. Report both
actual compressed sizes and headroom. Defer release rehearsal, preview/output
integration, and release documentation to the next phase.

## Phase release-integration

- Verify and complete the package configuration in `release-plz.toml` for the fixed
  versioning, tag, and GitHub Release policy. Preserve the existing daily guard, root
  changelog ownership, release triggers, OIDC workflow, and current rehearsal setting.
  Let release-plz own any new assets changelog; do not hand-author a release or bump
  either crate as an implementation step.
- Audit `.github/workflows/release.yml`, `tools/release/release_preview.py`,
  `release_outputs.py`, and their tests for single-package assumptions. Keep binary
  outputs selected by `package_name == bob-obsidian`. Test a mixed result with assets
  first, an assets-only result, and no release. Preserve root binstall URLs and binary
  tag selection. Update previews to report both package versions and relevant diffs if
  they otherwise hide assets changes.
- Rehearse with the pinned release-plz in a disposable fixture under the current
  workspace, with no push, tag publication, or registry mutation. Prove the
  configuration parses and updates work for a root-only change and an assets change that
  requires updating the consuming root dependency. Ensure assets and root cannot claim
  the same tag. Do not treat the globally installed 0.4.1 as this verification. Use
  existing repository history and disposable copies of the implementation for previews;
  do not create commits or retain release-generated version/changelog edits in the
  implementation.
- Update `docs/develop/releases/keep-the-package-under-the-size-cap.md` and
  `reference.md` to describe both packages, real post-split sizes, license ownership,
  per-package budgets, independent assets versions, and the shared train. Correct
  directly affected release preview/recovery instructions only. Document that future
  live mode needs Trusted Publisher setup for both crates; this task does not arm or
  switch the publisher.
- Run release-tool tests, `just docs-check`, and full `just package-check` after the
  final integration changes. Supply exact commands, results, measurements, and any
  environmental skips to the child epic land agent.

## Landing and completion

The child epic land agent checks the integrated change, resolves routine merge conflicts
with sibling parent phases, and runs the canonical `just check`,
`just check-release-tools`, `just check-web-clip-adapter`, `just docs-check`, and full
`just package-check`. Do not rerun unchanged passing suites without a new reason. Use
`/sase_monitor` for long commands, retaining responsibility for the handoff and eventual
result.

Acceptance requires byte-preserving migration, unchanged installed layout, working
direct-source consumers, successful full verification of both packages, both archives
below the warning threshold with recorded headroom, and evidence that the pinned release
tooling handles the new workspace without altering the root release contract. No
production publishing is part of acceptance.

A check failure caused by this split must be repaired. If it reproduces identically on
the clean base, record exact reproduction/evidence as `PROPOSED FOLLOW-UP:` on the
assigned bead, citing an existing task if known; it does not block completion. Do not
create task beads in this child epic. Pass all discovered follow-up entries to the
parent epic land agent for triage.

Before closing any assigned child phase, run `sase bead epic-symbols <id>` and resolve
entries or re-key them to a still-open owning bead. Planning inspection found no entries
for `bob-cli-da.4`, but the child land agent must check again.

Finally, append the verification evidence to `bob-cli-da.4`, run
`sase bead epic-symbols bob-cli-da.4`, clear or re-key all remaining entries, and run
`sase bead close bob-cli-da.4 --note "<before/after sizes and verified gates>"`. Do not
close `bob-cli-da`, `bob-cli-ai`, or any other ancestor plan. Report the task completion
evidence for the parent land agent to reconcile `bob-cli-ai`.
