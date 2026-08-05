<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.40.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.40.0** was hardened automatically. 1 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh writes the user-controlled `target` variable (sourced from `INPUT_TARGET` = `inputs.target`) and its derivatives (`target_upper`, `target_lower`) to `$GITHUB_ENV` in many places without the required newline-sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker supplying a target value containing embedded newlines could inject arbitrary environment variables into subsequent workflow steps.

Affected writes include:
- `printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${LD_LIBRARY_PATH:+...}" >> "${GITHUB_ENV}"` — also forwards the inherited `LD_LIBRARY_PATH` env var without sanitization.
- Multiple `cat >> "${GITHUB_ENV}" << EOF` heredocs embedding `${target_upper}` and `${target_lower}` (e.g. `CARGO_TARGET_${target_upper}_LINKER=...`, `CC_${target_lower}=...`).
- `printf '%s\n' "CARGO_TARGET_${target_upper}_RUNNER=${target}-runner" >> "${GITHUB_ENV}"`
- `printf '%s\n' "BINDGEN_EXTRA_CLANG_ARGS_${target_lower}=--sysroot=${sysroot_dir}" >> "${GITHUB_ENV}"`
- `printf '%s\n' "CARGO_TARGET_${target_upper}_RUNNER=qemu-${qemu_arch}" >> "${GITHUB_ENV}"`
- `printf 'CARGO_BUILD_TARGET=%s\n' "${target}" >> "${GITHUB_ENV}"`

None of these writes are preceded by the required sanitization: `safe=$(printf '%s' "$VAR" | tr -d '\n\r')`.

Locations:

- `main.sh:196`
- `main.sh:210`
- `main.sh:222`
- `main.sh:234`
- `main.sh:246`
- `main.sh:258`
- `main.sh:637`
- `main.sh:643`
- `main.sh:649`
- `main.sh:730`
- `main.sh:742`
- `main.sh:755`
- `main.sh:762`
- `main.sh:775`
- `main.sh:896`
- `main.sh:990`
- `main.sh:997`
- `main.sh:1003`
- `main.sh:1013`
- `main.sh:1185`
- `main.sh:1192`
- `main.sh:1197`
- `main.sh:1201`
- `action.yml:28`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Added sanitization of user-controlled `target`, `target_lower`, and `target_upper` variables in main.sh right after they are computed (after the `target_upper=$(tr ...)` line). The sanitization uses `printf '%s' "${var}" | tr -d '\n\r'` to strip embedded newlines and carriage returns. This single fix covers all 23 affected GITHUB_ENV write locations in main.sh since every write uses one of these three variables. The action.yml:28 finding is addressed by the main.sh fix - the `${{ inputs.target }}` expression in an `env:` block is the correct and safe pattern for passing inputs to composite action scripts.

