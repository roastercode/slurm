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
