<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.40.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.40.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. In main.sh, the user-controlled input `inputs.target` (passed as the `INPUT_TARGET` environment variable from action.yml) is read into the shell variable `target`, from which `target_upper`, `target_lower`, `cc_target`, `sysroot_dir`, `bin_dir`, and other derived variables are computed. All of these are written to `$GITHUB_ENV` and `$GITHUB_PATH` at numerous points throughout the script without the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`). For example:
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
index b2d5a75..b5b8214 100755
--- a/main.sh
+++ b/main.sh
@@ -36,7 +36,7 @@ normalize_comma_or_space_separated() {
   else
     # Otherwise, consider it is a whitespace-separated list.
     # Convert whitespace characters into comma.
-    sed -E 's/ +/,/g; s/^.//' <<<" ${list} "
+    sed -E 's/ +/,/g; s/^.//' <<< " ${list} "
   fi
 }
 _sudo() {
@@ -66,6 +66,8 @@ if [[ $# -gt 0 ]]; then
 fi
 
 target="${INPUT_TARGET:?}"
+# Sanitize target to prevent newline injection into GITHUB_ENV/GITHUB_PATH
+target=$(printf '%s' "${target}" | tr -d '\n\r')
 runner="${INPUT_RUNNER:-}"
 package="${INPUT_PACKAGE:-}"
 packages=()
@@ -247,7 +249,8 @@ install_rust_cross_toolchain() {
     riscv32gc-unknown-linux-gnu) sysroot_dir="${toolchain_dir}/sysroot" ;;
     loongarch64-unknown-linux-gnu)
       sysroot_dir="${toolchain_dir}/target/usr"
-      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${LD_LIBRARY_PATH:+":${LD_LIBRARY_PATH:-}"}" >>"${GITHUB_ENV}"
+      safe_ld_library_path=$(printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r')
+      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${safe_ld_library_path:+\":${safe_ld_library_path}\"}" >>"${GITHUB_ENV}"
       ;;
     *) sysroot_dir="${toolchain_dir}/${target}" ;;
   esac
@@ -669,12 +672,13 @@ CXX_${target_lower}=${apt_target}-g++
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

main.sh: `target` now has CR and LF stripped as soon as it is read, so `target_upper`, `target_lower`, `cc_target`, `sysroot_dir`, `bin_dir` and everything else built from it are safe before they reach GITHUB_ENV or GITHUB_PATH. The `runner` input gets the same treatment. Before inherited LD_LIBRARY_PATH and PKG_CONFIG_PATH are written to GITHUB_ENV, newlines are removed from them too. I re-applied the LD_LIBRARY_PATH change without the previous patch's escaped quotes (\"), which would have added literal quote characters to the path. In action.yml the previous harden only removed blank lines, which has no security effect, so I left that file alone. I also skipped the previous patch's `<<<` spacing change in main.sh, because it has no effect either.

