---
tier: tale
title: Fix Zsh startup dci alias collision
goal:
  New interactive Zsh shells load the chezmoi-managed aliases without a parse error,
  preserving the custom dci function.
size: small
proposed_by: bbugyi200.apollo.1u
create_time: 2026-09-26 06:17:06
status: wip
---

# Fix Zsh startup parse error from the `dci` alias collision

## Root cause

The chezmoi-managed `home/dot_zshrc` loads Oh My Zsh with the `docker` plugin before it
sources `home/dot_config/aliases.sh`. The plugin defines `dci='docker compose images'`.
The aliases file later declares `dci() { ...; }` at line 231. Zsh expands the existing
alias while parsing the function declaration, producing `parse error near '()'` and
stopping evaluation of the rest of the aliases file. `zsh -n` on the aliases file alone
passes because the collision depends on startup order. A fresh interactive shell
reproduces the error and reports `dci` as the plugin alias. An isolated shell that
removes that alias before sourcing the aliases file loads `dci` as a function without a
parse error.

## Implementation

1. Edit the chezmoi source `home/dot_zshrc`: add `dci` to the existing `bad_aliases`
   list, which is cleared immediately after Oh My Zsh loads and before `aliases.sh` is
   sourced. Keep the custom `dci()` function and the rest of startup behavior intact.
2. Inspect the resulting diff and verify no other edits are needed for this parse error.
3. After the source change is committed through the normal SASE finalization flow, run
   `chezmoi update -a --force` as required by the chezmoi repository instructions so the
   managed source reaches the home directory on this machine.

## Validation

1. Reproduce the collision in a clean Zsh process by defining an alias named `dci`
   before parsing a `dci()` declaration; confirm that removing the alias first succeeds.
2. Source the changed `home/dot_zshrc` startup sequence in a disposable Zsh process with
   the real Oh My Zsh plugin, and confirm `aliases.sh` loads without the line 231 parse
   error and `whence -w dci` reports a function.
3. Run `zsh -n` on the managed Zsh startup and aliases files. After chezmoi applies the
   committed source, run a fresh interactive Zsh smoke check on this machine and confirm
   the parse error is gone and `dci` resolves to the custom function. Ignore unrelated
   non-TTY `stty` noise in noninteractive probes.

## Scope

This change addresses the alias/function name collision in the chezmoi source and
applies it to the affected machine. It does not alter the custom `dci` function body or
the Docker plugin's other aliases.
