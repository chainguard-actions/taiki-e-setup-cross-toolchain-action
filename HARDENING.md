<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.41.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.41.0** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh writes the `target` variable (sourced from INPUT_TARGET / inputs.target) and variables derived from it (`target_upper`, `target_lower`, `sysroot_dir`, `cc_target`) to $GITHUB_ENV in many places using both `printf` and `cat` heredocs, without any sanitization step (`tr -d '\n\r'`). Because action.yml maps `inputs.target` directly to the INPUT_TARGET env var with no validation, a caller supplying a value containing embedded newlines could inject arbitrary key=value pairs into the runner's environment. Representative unsanitized writes include:
- `printf 'CARGO_BUILD_TARGET=%s\n' "${target}" >> "${GITHUB_ENV}"`
- `printf '%s\n' "CARGO_TARGET_${target_upper}_RUNNER=${target}-runner" >> "${GITHUB_ENV}"`
- `printf 'LD_LIBRARY_PATH=...${target}/lib...' >> "${GITHUB_ENV}"`
- `printf '%s\n' "BINDGEN_EXTRA_CLANG_ARGS_${target_lower}=--sysroot=${sysroot_dir}" >> "${GITHUB_ENV}"`
- `printf 'QEMU_LD_PREFIX=%s\n' "${sysroot_dir}" >> "${GITHUB_ENV}"`
- Multiple `cat >> "${GITHUB_ENV}" <<EOF` heredocs expanding `${target_upper}`, `${target_lower}`, `${target}`
None of these writes are preceded by the required sanitization pipeline `printf '%s' "$VAR" | tr -d '\n\r'`.

Locations:

- `action.yml:29`
- `main.sh:209`
- `main.sh:218`
- `main.sh:228`
- `main.sh:238`
- `main.sh:248`
- `main.sh:637`
- `main.sh:643`
- `main.sh:649`
- `main.sh:735`
- `main.sh:748`
- `main.sh:762`
- `main.sh:912`
- `main.sh:1003`
- `main.sh:1036`
- `main.sh:1198`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed GITHUB_ENV injection vulnerability in main.sh by sanitizing the `target` variable (sourced from INPUT_TARGET) at the point of assignment, before any derived variables are computed. Added `target=$(printf '%s' "${target}" | tr -d '\n\r')` immediately after `target="${INPUT_TARGET:?}"` and again after the `host-tuple` substitution. Since all variables written to GITHUB_ENV (target_upper, target_lower, sysroot_dir, cc_target, apt_target) are derived from `target`, this single fix at the root covers all 16 vulnerable write locations identified in the finding.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed two github-env-injection vulnerabilities in main.sh:
1. LD_LIBRARY_PATH (line ~228, loongarch64 case): Sanitized the inherited $LD_LIBRARY_PATH value using `safe_ld_library_path=$(printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r')` before embedding it in the GITHUB_ENV printf statement.
2. PKG_CONFIG_PATH (line ~604, linux-gnu heredoc): Sanitized the inherited $PKG_CONFIG_PATH value using `safe_pkg_config_path=$(printf '%s' "${PKG_CONFIG_PATH:-}" | tr -d '\n\r')` before the heredoc, and replaced the direct variable reference in the heredoc with the sanitized variable. Both fixes prevent a calling workflow from injecting arbitrary key=value pairs into the runner environment via newlines in these inherited environment variables.

