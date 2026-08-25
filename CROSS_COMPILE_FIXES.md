# Slurm Cross-Compilation Fixes for Yocto/OpenEmbedded

## Problem

When cross-compiling Slurm 23.11.x with Yocto/OpenEmbedded (Scarthgap),
plugins fail to load at runtime with errors like:

slurmd: error: plugin_load_from_file: dlopen(/usr/lib/slurm/select_cons_tres.so):
undefined symbol: license_copy
slurmd: error: plugin_load_from_file: dlopen(/usr/lib/slurm/topology_default.so):
undefined symbol: unlock_slurmctld


## Root Cause

Slurm plugins reference internal symbols from `slurmctld`/`slurmd` binaries
that are not exported by `libslurm.so`. In native builds the dynamic linker
resolves them at runtime. Cross-compilation toolchains enforce symbol
resolution at link time, causing failures.

## Fixes Applied (this branch)

| Patch | Description |
|-------|-------------|
| 0001 | `-Wl,--allow-shlib-undefined` on all plugin Makefiles |
| 0002 | `DEFAULT_PREP_PLUGINS=""` — prep/script uses slurmd-internal symbol |
| 0003 | `-Wl,--export-dynamic` on slurmctld |
| 0004 | `RTLD_LAZY\|RTLD_GLOBAL` in plugin.c |
| 0005 | Symbol stubs in libslurmfull for client tools |

## Testing

Tested on Yocto Scarthgap, cross-compile x86_64-poky-linux, KVM/QEMU:
- slurmctld starts correctly
- slurmd registers 3 compute nodes
- sinfo shows nodes idle
- sbatch submits and executes jobs

## Yocto Integration

Meta-layer with Yocto recipes and patches:
https://github.com/roastercode/yocto-hardened/tree/yocto-hpc

## Upstream Report

Bug filed at: https://support.schedmd.com/ (pending)

## Additional Issue: lz4/json/yaml Detection Under Poisoned System Directories

### Problem

When cross-compiling Slurm 25.11.x under GCC 16+ hosts (observed during
a Yocto styhead->walnascar migration, 2026-08-25), `do_compile` fails
with:

cc1: error: include location "/usr/include" is unsafe for cross-compilation [-Werror=poison-system-directories]


### Root Cause

`auxdir/x_ac_lz4.m4`, `x_ac_json.m4`, and `x_ac_yaml.m4` default to
probing hardcoded paths (`/usr/local /usr /opt/local /sw`) when no
`--with-lz4`/`--with-json`/`--with-yaml` is given. In a Yocto
cross-compile sysroot, these libraries' headers are staged under the
sysroot's own `/usr/include`, but the probe finds a match on plain
`/usr` from its search list without going through the sysroot path,
producing a bare `-I/usr/include` on the compile line -- which the
target GCC's `-Werror=poison-system-directories` check (present since
GCC 14, enforced more consistently with GCC 16 host toolchains)
correctly rejects as a build-host directory leaking into a
cross-compiled binary.

This is a detection-path issue, not a symbol-resolution one, so it is
distinct from patches 0001-0005 above and does not require a source
patch to slurm itself.

### Fix (recipe-level, not a source patch)

Pass explicit paths, the same way `--with-munge`/`--with-pmix`/
`--with-hwloc` already must be:

--with-lz4=${STAGING_DIR_TARGET}${prefix}
--with-json=${STAGING_DIR_TARGET}${prefix}
--with-yaml=${STAGING_DIR_TARGET}${prefix}


Applied in the beamfs-lab Yocto layer's `slurm_25.11.4.bb`
(`EXTRA_OECONF`), not in this fork, since it requires no change to
slurm's own source.
