<!-- markdownlint-disable -->

# Hardening Report: taiki-e--setup-cross-toolchain-action/v1.42.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--setup-cross-toolchain-action/v1.42.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. main.sh writes values derived from user-controlled inputs (INPUT_TARGET, INPUT_RUNNER, INPUT_QEMU, INPUT_VALGRIND, INPUT_WINE — all set from `inputs.*` in action.yml) into $GITHUB_ENV via heredoc (`cat >> "${GITHUB_ENV}" <<EOF`) and `printf ... >> "${GITHUB_ENV}"` without applying the required newline-stripping sanitization (`printf '%s' "$VAR" | tr -d '\n\r'`). Variables such as `target`, `target_upper`, `target_lower`, `cc_target`, `api_level`, `apt_target`, `sysroot_dir`, `bin_dir`, `qemu_arch`, and `default_qemu_cpu` are all derived from these inputs and written unsanitized. An attacker supplying a `target` input containing embedded newlines (e.g. `x86_64-unknown-linux-gnu\nMALICIOUS_VAR=evil`) could inject arbitrary environment variables into subsequent workflow steps.

Locations:

- `main.sh:1`
- `action.yml:28`

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
index c988879..075de4d 100755
--- a/main.sh
+++ b/main.sh
@@ -65,9 +65,15 @@ if [[ $# -gt 0 ]]; then
   bail "invalid argument '$1'"
 fi
 
-target="${INPUT_TARGET:?}"
-runner="${INPUT_RUNNER:-}"
-package="${INPUT_PACKAGE:-}"
+# Sanitize user-controlled inputs by stripping newlines/carriage returns to
+# prevent GITHUB_ENV injection attacks (CWE-93 / github-env-injection).
+sanitize_input() {
+  printf '%s' "$1" | tr -d '\n\r'
+}
+
+target="$(sanitize_input "${INPUT_TARGET:?}")"
+runner="$(sanitize_input "${INPUT_RUNNER:-}")"
+package="$(sanitize_input "${INPUT_PACKAGE:-}")"
 packages=()
 if [[ -n "${package}" ]]; then
   while IFS= read -rd,; do
@@ -247,7 +253,8 @@ install_rust_cross_toolchain() {
     riscv32gc-unknown-linux-gnu) sysroot_dir="${toolchain_dir}/sysroot" ;;
     loongarch64-unknown-linux-gnu)
       sysroot_dir="${toolchain_dir}/target/usr"
-      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${LD_LIBRARY_PATH:+":${LD_LIBRARY_PATH:-}"}" >>"${GITHUB_ENV}"
+      safe_ld_library_path="$(printf '%s' "${LD_LIBRARY_PATH:-}" | tr -d '\n\r')"
+      printf 'LD_LIBRARY_PATH=%s\n' "${toolchain_dir}/target/usr/lib64:${toolchain_dir}/${target}/lib${safe_ld_library_path:+":${safe_ld_library_path}"}" >>"${GITHUB_ENV}"
       ;;
     *) sysroot_dir="${toolchain_dir}/${target}" ;;
   esac
@@ -669,12 +676,13 @@ CXX_${target_lower}=${apt_target}-g++
 EOF
                 ;;
             esac
+            safe_pkg_config_path="$(printf '%s' "${PKG_CONFIG_PATH:-}" | tr -d '\n\r')"
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
@@ -824,7 +832,7 @@ EOF
                   *) bail "unrecognized runner '${runner}'" ;;
                 esac
                 _sudo cp -- "${wine_root}"/lib/ld-linux-aarch64.so.1 /lib/
-                qemu_version="${INPUT_QEMU:-"${default_qemu_version}"}"
+                qemu_version="$(sanitize_input "${INPUT_QEMU:-"${default_qemu_version}"}")"
                 install_qemu
                 x "${wine_exe}" --version
                 wineboot="${wine_root}/bin/wineserver"
@@ -838,7 +846,7 @@ EOF
                 version_id=$(grep -E '^VERSION_ID=' /etc/os-release | cut -d= -f2)
                 version_id="${version_id//\"/}"
                 case "${runner}" in
-                  '' | wine) wine_version="${INPUT_WINE:-"${default_wine_version}"}" ;;
+                  '' | wine) wine_version="$(sanitize_input "${INPUT_WINE:-"${default_wine_version}"}")" ;;
                   wine@*) wine_version="${runner#*@}" ;;
                   *) bail "unrecognized runner '${runner}'" ;;
                 esac
@@ -1110,8 +1118,8 @@ case "${host}" in
   *-linux-gnu*)
     use_qemu=''
     use_valgrind=''
-    qemu_version="${INPUT_QEMU:-"${default_qemu_version}"}"
-    valgrind_version="${INPUT_VALGRIND:-"${default_valgrind_version}"}"
+    qemu_version="$(sanitize_input "${INPUT_QEMU:-"${default_qemu_version}"}")"
+    valgrind_version="$(sanitize_input "${INPUT_VALGRIND:-"${default_valgrind_version}"}")"
     case "${target}" in
       *-linux-* | *-android*)
         case "${runner}" in

```

Locations:

- `main.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

main.sh: CR and LF characters are now stripped from INPUT_TARGET, INPUT_RUNNER, INPUT_QEMU, INPUT_VALGRIND and INPUT_WINE as soon as they are read. Every value that ends up in $GITHUB_ENV is built from these inputs (target, target_upper, target_lower, apt_target, sysroot_dir, api_level, the qemu/valgrind/wine versions, the wine@ runner suffix). I also strip them from the inherited LD_LIBRARY_PATH and PKG_CONFIG_PATH before those are written to $GITHUB_ENV. The stripping uses bash parameter expansion (${v//[$'\r\n']/}) rather than the previous patch's $(sanitize_input ...) subshells. With the subshells, a failure inside the substitution wouldn't stop the script under errexit; the expansion can't hide a failure that way. INPUT_PACKAGE is deliberately left alone: newlines separate packages in that list (normalize_comma_or_space_separated), the values only go to apt install and are never written to $GITHUB_ENV, and stripping newlines would join separate package names into one. action.yml is unchanged: the earlier patch to it only deleted blank lines and had no security effect. Nothing was run or tested in this pass.

