<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.43.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.43.1** was hardened automatically. 1 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh is the supporting script run by the action's only composite `run:` step. It writes values built from the workflow-controlled input `inputs.target` (passed as INPUT_TARGET and then copied into `target`) to $GITHUB_ENV, and nothing strips newlines first (`printf '%s' ... | tr -d '\n\r'`). The plainest case is the last line of the script: `printf 'CARGO_BUILD_TARGET=%s\n' "${target}" >>"${GITHUB_ENV}"`. Other writes do the same thing: the `cat >>"${GITHUB_ENV}" <<EOF` blocks that use `${target}`, `${target_upper}` and `${target_lower}`; `BINDGEN_EXTRA_CLANG_ARGS_${target_lower}=...`; and `CARGO_TARGET_${target_upper}_RUNNER=...`. The loongarch64 branch also forwards the inherited `LD_LIBRARY_PATH` variable unsanitized. The target is only checked with glob `case` patterns such as `x86_64*`, and bash's `*` also matches newlines. So a target value containing a newline could inject extra `NAME=value` lines (for example LD_PRELOAD or PATH changes) into the environment of later steps. Fix: strip newlines first, e.g. `target=$(printf '%s' "$INPUT_TARGET" | tr -d '\n\r')`, or reject targets that don't match `^[A-Za-z0-9_.@-]+$`.

Locations:

- `main.sh:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

main.sh: the target input is now checked as soon as it is read from INPUT_TARGET. Anything that does not match ^[A-Za-z0-9_.@-]+$ stops the script with an error. This covers every target triple plus the @version suffix, and it means no target can carry a newline into the many places it is written to GITHUB_ENV: CARGO_BUILD_TARGET, the CARGO_TARGET_*_RUNNER and BINDGEN_EXTRA_CLANG_ARGS_* lines, and the cat-heredoc blocks. The 'host-tuple' value still passes the check, so that special case works as before. On the loongarch64 path, the LD_LIBRARY_PATH value (built from the inherited variable) now has CR and LF removed before it is written to GITHUB_ENV. Nothing was run or tested. Risk: a target containing any character outside that set, such as a path to a custom target .json file, would now be refused; I did not check whether upstream supports those.

