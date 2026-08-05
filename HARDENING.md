<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.40.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.40.1** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In main.sh, the user-controlled input `inputs.target` (passed as the `INPUT_TARGET` environment variable from action.yml) is read into the shell variable `target`, from which `target_upper`, `target_lower`, `cc_target`, `sysroot_dir`, `bin_dir`, and other derived variables are computed. All of these are written to `$GITHUB_ENV` and `$GITHUB_PATH` at numerous points throughout the script without the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`). For example:
- `printf 'CARGO_BUILD_TARGET=%s\n' "${target}" >> "${GITHUB_ENV}"` (line ~49341)
- `printf 'LD_LIBRARY_PATH=%s\n' ".../${target}/lib..." >> "${GITHUB_ENV}"` (line ~8778)
- `printf '%s\n' "CARGO_TARGET_${target_upper}_RUNNER=..." >> "${GITHUB_ENV}"` (line ~41235)
- `printf '%s\n' "${bin_dir}" >> "${GITHUB_PATH}"` (line ~48648)
- Multiple `cat >> "${GITHUB_ENV}" <<EOF` heredocs embedding `${target_upper}`, `${target_lower}`, `${cc_target}`, `${sysroot_dir}` (lines ~9094, ~9479, ~9955, ~10320, ~10651, ~26440, ~26641, ~26850, ~30202, ~30735, ~30887, ~31460, ~31940, ~37756)

An attacker who controls the `target` input (e.g. via `workflow_dispatch` or by calling this composite action from a workflow) can embed newline characters to inject arbitrary key=value pairs into `$GITHUB_ENV`, potentially overriding sensitive environment variables for subsequent steps.

Locations:

- `main.sh:1601`
- `main.sh:3808`
- `main.sh:3809`
- `main.sh:3810`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Added newline sanitization for the `target` variable in main.sh immediately after it is set from `INPUT_TARGET`. The fix uses `target=$(printf '%s' "${target}" | tr -d '\n\r')` to strip all newline and carriage return characters. Since all derived variables (`target_lower`, `target_upper`, `cc_target`, `sysroot_dir`, and path components) are computed from `target` via shell parameter expansion, sanitizing the source variable prevents injection across all 26+ writes to `$GITHUB_ENV` and the write to `$GITHUB_PATH` throughout the script.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection vulnerabilities in main.sh:
1. Line ~228 (LD_LIBRARY_PATH): Added `safe_ld_library_path=$(printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r')` before the printf that writes to $GITHUB_ENV, and replaced `${LD_LIBRARY_PATH:-}` with `${safe_ld_library_path}` in the printf format.
2. Line ~620 (PKG_CONFIG_PATH): Added `safe_pkg_config_path=$(printf '%s' "${PKG_CONFIG_PATH:-}" | tr -d '\n\r')` before the heredoc that writes to $GITHUB_ENV, and replaced `${PKG_CONFIG_PATH:-}` with `${safe_pkg_config_path}` in the heredoc body.
Both fixes strip newline and carriage return characters from inherited environment variables before writing them to $GITHUB_ENV, preventing injection of arbitrary environment variables.

