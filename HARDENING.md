<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.42.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.42.0** was hardened automatically. 1 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh writes values derived from user-controlled inputs (INPUT_TARGET, INPUT_RUNNER, INPUT_QEMU, INPUT_VALGRIND, INPUT_WINE — all set from `inputs.*` in action.yml) into $GITHUB_ENV via heredoc (`cat >> "${GITHUB_ENV}" <<EOF`) and `printf ... >> "${GITHUB_ENV}"` without applying the required newline-stripping sanitization (`printf '%s' "$VAR" | tr -d '\n\r'`). Variables such as `target`, `target_upper`, `target_lower`, `cc_target`, `api_level`, `apt_target`, `sysroot_dir`, `bin_dir`, `qemu_arch`, and `default_qemu_cpu` are all derived from these inputs and written unsanitized. An attacker supplying a `target` input containing embedded newlines (e.g. `x86_64-unknown-linux-gnu\nMALICIOUS_VAR=evil`) could inject arbitrary environment variables into subsequent workflow steps.

Locations:

- `main.sh:1`
- `action.yml:28`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Added sanitize_input() function using 'printf "%s" "$1" | tr -d "\n\r"' to strip newlines/carriage returns from all user-controlled inputs. Applied sanitization to: INPUT_TARGET→target (primary attack vector; all derived vars like target_upper, target_lower, cc_target, apt_target, sysroot_dir, qemu_arch, default_qemu_cpu inherit the sanitization), INPUT_RUNNER→runner (derived qemu_version/valgrind_version/wine_version via ${runner#*@} are also clean), INPUT_PACKAGE→package, and INPUT_QEMU/INPUT_VALGRIND/INPUT_WINE at all their assignment points. This prevents an attacker from injecting arbitrary environment variables into subsequent workflow steps via embedded newlines in the target input.

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed the github-env-injection vulnerability in main.sh at the loongarch64-unknown-linux-gnu case inside install_rust_cross_toolchain(). The inherited LD_LIBRARY_PATH environment variable was being written directly to $GITHUB_ENV without sanitization, allowing a calling workflow to inject additional key=value pairs via newline characters. The fix captures LD_LIBRARY_PATH into a sanitized local variable `safe_ld_library_path` using `printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r'` before using it in the printf statement that writes to $GITHUB_ENV. The conditional expansion logic is preserved so the path is only appended when non-empty.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed github-env-injection in hardened/action/main.sh: added `safe_pkg_config_path="$(printf '%s' "${PKG_CONFIG_PATH:-}" | tr -d '\n\r')"` before the heredoc that writes to $GITHUB_ENV, and replaced the unsanitized `${PKG_CONFIG_PATH:-}` reference in the heredoc with `${safe_pkg_config_path}`. This prevents a calling workflow from injecting additional KEY=VALUE pairs into the GitHub Actions environment file via embedded newlines in PKG_CONFIG_PATH. The fix follows the same pattern already used for LD_LIBRARY_PATH in the loongarch64 case.

