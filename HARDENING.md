<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.39.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.39.2** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh writes user-controlled input values to $GITHUB_ENV and $GITHUB_PATH without newline sanitization. The `target` variable is set directly from INPUT_TARGET (mapped from inputs.target in action.yml), and derived variables (target_upper, target_lower, cc_target, api_level, sysroot_dir, qemu_arch, bin_dir, etc.) are all written to $GITHUB_ENV/$GITHUB_PATH in many places via `printf '%s\n' "...${target}..." >> "${GITHUB_ENV}"` and `cat >> "${GITHUB_ENV}" <<EOF ... ${target_upper} ... EOF` heredocs. Similarly, INPUT_QEMU feeds qemu_version which influences values written to GITHUB_ENV. None of these writes are preceded by the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`). An attacker supplying a newline-containing value for inputs.target could inject arbitrary environment variables into subsequent steps.

Locations:

- `main.sh:196`
- `main.sh:213`
- `main.sh:228`
- `main.sh:244`
- `main.sh:258`
- `main.sh:272`
- `main.sh:476`
- `main.sh:484`
- `main.sh:492`
- `main.sh:556`
- `main.sh:569`
- `main.sh:574`
- `main.sh:591`
- `main.sh:601`
- `main.sh:720`
- `main.sh:808`
- `main.sh:811`
- `main.sh:815`
- `main.sh:820`
- `main.sh:843`
- `main.sh:845`
- `main.sh:848`
- `main.sh:850`
- `main.sh:980`
- `main.sh:983`
- `main.sh:985`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Added newline/carriage-return sanitization for user-controlled inputs that flow into $GITHUB_ENV and $GITHUB_PATH writes in main.sh:
1. Sanitized `target` (from INPUT_TARGET) immediately after assignment: `target=$(printf '%s' "${target}" | tr -d '\n\r')`. This fixes all 26 downstream write locations since target_upper, target_lower, cc_target, sysroot_dir, qemu_arch, and all other derived variables come from `target`.
2. Sanitized `qemu_version` (from INPUT_QEMU) at both assignment points: inside the wine/aarch64 Windows runner block and in the main *-linux-gnu* host case block.

### Iteration 2

**Fixes applied:** script-injection, unpinned-uses

**Notes:**

Fixed all findings in ci.yml and release.yml:

1. script-injection (3 locations in ci.yml lines 246, 340, 374): Moved `${{ matrix.target }}` out of `run:` shell strings into `env:` blocks as `MATRIX_TARGET: ${{ matrix.target }}`, then referenced as `"$MATRIX_TARGET"` in the shell commands for all three jobs (test, test-container, test-valgrind).

2. unpinned-uses (4 locations):
   - ci.yml line 36: `taiki-e/github-actions/.github/workflows/tidy.yml@main` → pinned to SHA `949aa0f9e198f5cf3b32b9721f4612904b08939b # main`
   - ci.yml lines 228-229 (×3 jobs): `taiki-e/checkout-action@v1` → pinned to SHA `7d1e50e93dc4fb3bba58f85018fadf77898aee8b # v1`; `taiki-e/github-actions/install-rust@nightly` → pinned to SHA `949aa0f9e198f5cf3b32b9721f4612904b08939b # nightly`
   - release.yml line 23: `taiki-e/github-actions/.github/workflows/action-release.yml@main` → pinned to SHA `949aa0f9e198f5cf3b32b9721f4612904b08939b # main`

