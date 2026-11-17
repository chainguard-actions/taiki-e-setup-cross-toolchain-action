<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.41.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.41.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. main.sh writes the `target` variable (sourced from INPUT_TARGET / inputs.target) and variables derived from it (`target_upper`, `target_lower`, `sysroot_dir`, `cc_target`) to $GITHUB_ENV in many places using both `printf` and `cat` heredocs, without any sanitization step (`tr -d '\n\r'`). Because action.yml maps `inputs.target` directly to the INPUT_TARGET env var with no validation, a caller supplying a value containing embedded newlines could inject arbitrary key=value pairs into the runner's environment. Representative unsanitized writes include:
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

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 2fbe02b..825a541 100644
--- a/action.yml
+++ b/action.yml
@@ -1,6 +1,5 @@
 name: Setup toolchains for cross compilation and cross testing for Rust
 description: GitHub Action for setup toolchains for cross compilation and cross testing for Rust
-
 inputs:
   target:
     description: Target name
@@ -20,13 +19,29 @@ inputs:
   package:
     description: Packages to install for target (whitespace or comma separated list)
     required: false
-
 # Note:
 # - inputs.* should be manually mapped to INPUT_* due to https://github.com/actions/runner/issues/665
 # - For context containing path, use GITHUB_*/RUNNER_* instead of github.*/runner.* due to https://github.com/actions/runner/issues/2185
 runs:
   using: composite
   steps:
     - run: bash --noprofile --norc "${GITHUB_ACTION_PATH:?}/main.sh"
       shell: bash
       env:

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed main.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/main.sh b/main.sh
index ee7b778..356baac 100755
--- a/main.sh
+++ b/main.sh
@@ -66,6 +66,8 @@ if [[ $# -gt 0 ]]; then
 fi
 
 target="${INPUT_TARGET:?}"
+# Sanitize target: strip newlines/carriage returns to prevent GITHUB_ENV injection
+target=$(printf '%s' "${target}" | tr -d '\n\r')
 runner="${INPUT_RUNNER:-}"
 package="${INPUT_PACKAGE:-}"
 packages=()
@@ -79,6 +81,8 @@ host=$(rustc -vV | grep -E '^host:' | cut -d' ' -f2)
 if [[ "${target}" == "host-tuple" ]]; then
   target="${host}"
 fi
+# Re-sanitize after potential host-tuple substitution (host is from rustc, not user input, but be safe)
+target=$(printf '%s' "${target}" | tr -d '\n\r')
 rustc_version=$(rustc -vV | grep -E '^release:' | cut -d' ' -f2)
 rustc_minor_version="${rustc_version#*.}"
 rustc_minor_version="${rustc_minor_version%%.*}"
@@ -247,7 +251,8 @@ install_rust_cross_toolchain() {
     riscv32gc-unknown-linux-gnu) sysroot_dir="${toolchain_dir}/sysroot" ;;
     loongarch64-unknown-linux-gnu)
       sysroot_dir="${toolchain_dir}/target/usr"
-      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${LD_LIBRARY_PATH:+":${LD_LIBRARY_PATH:-}"}" >>"${GITHUB_ENV}"
+      safe_ld_library_path=$(printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r')
+      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${safe_ld_library_path:+":${safe_ld_library_path}"}" >>"${GITHUB_ENV}"
       ;;
     *) sysroot_dir="${toolchain_dir}/${target}" ;;
   esac
@@ -669,12 +674,13 @@ CXX_${target_lower}=${apt_target}-g++
 EOF
                 ;;
             esac
+            safe_pkg_config_path=$(printf '%s' "${PKG_CONFIG_PATH:-}" | tr -d '\n\r')
             cat >>"${GITHUB_ENV}" <<EOF
 AR_${target_lower}=${apt_target}-ar
 RANLIB_${target_lower}=${apt_target}-ranlib
 STRIP=${apt_target}-strip
 OBJDUMP=${apt_target}-objdump
-PKG_CONFIG_PATH=/usr/lib/${apt_target}/pkgconfig:${PKG_CONFIG_PATH:-}
+PKG_CONFIG_PATH=/usr/lib/${apt_target}/pkgconfig:${safe_pkg_config_path}
 PKG_CONFIG_SYSROOT_DIR=${sysroot_dir}
 EOF
             ;;

```

Locations:

- `main.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

main.sh: target is now cleaned with tr -d '\n\r' right after it is read from INPUT_TARGET, and cleaned again after the host-tuple substitution. target_upper, target_lower, sysroot_dir, cc_target and the other values written to GITHUB_ENV are all built from target, so cleaning it once covers every write the finding lists. Re-applied the previous harden's other two fixes: inherited LD_LIBRARY_PATH (loongarch64 branch) and PKG_CONFIG_PATH (apt cross-compile branch) now have newlines removed before they go into GITHUB_ENV. The previous action.yml patch only removed blank lines and changed nothing security-related, so I left action.yml as it was: inputs already go to env vars and there is no inline expression in run:. I didn't run anything to test these edits.

