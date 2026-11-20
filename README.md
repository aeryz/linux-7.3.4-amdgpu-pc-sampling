# linux-7.3.4-amdgpu-pc-sampling

The AMD GPU driver (`amdgpu` with `amdkfd`) from Linux 7.3.4, with KFD PC sampling (host trap and
stochastic) for RDNA 3.5 APUs: gfx1150 (Strix Point, e.g. Radeon 890M) and gfx1151 (Strix Halo).
rocprofiler-sdk can attribute the samples to their kernel dispatch in hardware.

## Credit

The PC sampling support is the work of **[woct0rdho](https://github.com/woct0rdho)**, who brought
it up on gfx1151:

- driver: [woct0rdho/linux-amdgpu-driver](https://github.com/woct0rdho/linux-amdgpu-driver), branch
  `pc_sampling_gfx1151` (at `2a44424`)
- userspace: [woct0rdho/rocm-systems](https://github.com/woct0rdho/rocm-systems), branch
  `pc_sampling_gfx1151`
- discussion: [ROCm/rocm-systems#3428](https://github.com/ROCm/rocm-systems/issues/3428)

That work in turn builds on AMD's KFD PC sampling implementation (see the header of
`drivers/gpu/drm/amd/amdkfd/kfd_pc_sampling.c`). This repository only ports it to Linux 7.3.4,
enables it on gfx1150, and works around a firmware behavior that broke dispatch correlation.

## Contents

Besides this README, only `drivers/gpu/drm/amd/` and `include/uapi/linux/kfd_ioctl.h`, at their
kernel-tree paths, so `git diff f27a3b8 HEAD -- drivers include` is a patch for a full Linux 7.3.4
tree (`patch -p1`).

| Commit | Change |
|---|---|
| `f27a3b8` | Stock Linux 7.3.4 sources, unmodified |
| `24cf980` | woct0rdho's PC sampling patch ported to 7.3.4 (CWSR trap flags through `iosys_map`, PC sampling ioctl at 0x85 next to 7.3's `AMDKFD_IOC_PROFILER`), plus GC 11.5.0 (gfx1150) enabled |
| `b12c433` | Keeps `SPI_GDBG_PER_VMID_CNTL.TRAP_EN` set while sampling. MES firmware keeps clearing it, and without it SPI doesn't write the dispatch index, workgroup ID and wave ID into a wave's trap temporaries, so no sample can be tied to its dispatch. A delayed work re-sets it for all KFD VMIDs every 1 ms during sampling sessions, and GFXOFF is held off meanwhile |

## Building

The module must be built for exactly the kernel it is loaded into: Linux 7.3.4 with the same
config. On NixOS the module is built by a derivation that follows these steps (the only way this
was tested):

```bash
cd linux-7.3.4                                  # full source tree of the running kernel
git -C /path/to/linux-7.3.4-amdgpu-pc-sampling diff f27a3b8 HEAD -- drivers include | patch -p1
cp /lib/modules/$(uname -r)/build/.config /lib/modules/$(uname -r)/build/Module.symvers .
make olddefconfig                               # must leave .config unchanged
make modules_prepare
ln -s /path/to/running/vmlinux vmlinux          # module BTF is generated against it
make M=drivers/gpu/drm/amd/amdgpu modules
sudo install -Dm644 drivers/gpu/drm/amd/amdgpu/amdgpu.ko /lib/modules/$(uname -r)/updates/amdgpu.ko
sudo depmod -a
```

If your initramfs includes `amdgpu`, rebuild it too. With the module loaded, the KFD reports PC
sampling capabilities, e.g. `rocprofv3-avail list --pc-sampling` lists the GPU under "Agents
supporting PC Sampling".

Userspace needs woct0rdho's rocm-systems fork above. For gfx1150, rocprofiler-sdk and kfdtest
additionally have to accept `gfx1150` (they only check for gfx1151), and per-workgroup/wave
attribution needs a gfx11 sampling trap handler in ROCr that preserves `ttmp8`–`ttmp11`.

## Results

Radeon 890M (gfx1150, Ryzen AI 9 HX 370), rocprofv3 with `--kernel-trace`, samples that
rocprofiler-sdk attributed to their dispatch from the hardware correlation id:

| Method | Without `b12c433` | With `b12c433` |
|---|---|---|
| host trap | 0 / 3,181 | 2,805 / 2,835 |
| stochastic | 0 / 96,468 | 80,171 / 80,873 |

Every attributed sample's timestamp lies inside its dispatch, and its PC inside the dispatch's
kernel.

## Limitations

- Waves launched in the up-to-1 ms gap between MES clearing `TRAP_EN` and the next re-set carry
  no dispatch index (about 1% of samples above).
- GFXOFF stays disabled while any PC sampling session is running, which costs idle power.
- Stochastic samples take their PC from the shared `SQ_PERF_SNAPSHOT_*` registers but their
  dispatch and wave identity from the trapping wave. Under concurrent load 0.08–0.9% of them carried
  the PC of another wave on the same SIMD. Host-trap samples are unaffected.
- The PC sampling ioctl number (0x85) is not an upstream ABI.
- Tested only on gfx1150 with Linux 7.3.4 on NixOS. `b12c433` applies to all GC 11 GPUs, gfx1151
  included, but this port was not tested on gfx1151.

## License

Files keep the license headers they have in Linux (GPL-2.0 and/or MIT, see each file's SPDX line).
