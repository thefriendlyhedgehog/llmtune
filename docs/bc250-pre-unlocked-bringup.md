# llmtune on a board that is already unlocked (MastaG kernel, Limine, unlocked firmware)

[bc250-complete-bringup.md](bc250-complete-bringup.md) unlocks the board with arieltune: a pinned
`linux-cachyos-bore` 7.0.9-1 kernel built by `aputune build`, GRUB, and the 8-core unlock from the TUI.
This page is for a board that already has 8 cores and 40 CUs another way:

* unlocked UEFI (MeiMei/Forbidden-Darkness: Core Unlock + SMU Unlock + SMU Patch), UMA 512M
* the MastaG `linux-cachyos-bc250` kernel from the `bc250-cachyos` pacman repo, 40 CU via
  `options amdgpu bc250_cc_write_mode=3`
* Limine as the bootloader
* `cyan-skillfish-governor-smu`, GPU capped at 1500 MHz

On that board, **skip steps 2 to 10 of the complete guide**. Steps 3, 6 and 8 would replace the working
7.2.x kernel with a 7.0.9 one and pin it, and the two kernels carry different patch sets. Step 5 edits
GRUB, which a Limine install does not use. Go straight to llmtune, with the changes below.

## 1. Check what the running kernel already has

These only read state:

```sh
uname -r                                              # expect ...-cachyos-bc250
cat /proc/cmdline
ls /sys/module/amdgpu/parameters/ | grep bc250        # bc250_cc_write_mode, bc250_flush_by_runlist, ...
cat /sys/module/amdgpu/parameters/bc250_cc_write_mode # 3
cat /sys/module/amdgpu/parameters/gpu_recovery        # -1 = auto (default)
grep -h simd_count /sys/class/kfd/kfd/topology/nodes/*/properties   # 80 = 40 CU
```

## 2. Kernel command line: what carries over and what does not

The arieltune baseline line (`arieltune/README.md`, "Arm the fleet kernel command line") mixes flags
for the arieltune kernel, ROCm, and the Vulkan inference path. For llmtune, which serves through Vulkan
(RADV), only some of them matter:

| flag from the arieltune line | on the MastaG kernel serving Vulkan | why |
|---|---|---|
| `ttm.pages_limit=3588867 ttm.page_pool_size=3588867` | **use it** | raises the GPU-reachable system memory to ~13.7 GiB. llmtune's profiles size their contexts for ~14 GiB of UMA (`profiles.toml`), and a stock limit leaves less (12019 MiB measured on one board with `llama-cli --list-devices`) |
| `amdgpu.gpu_recovery=1` | **use `=0` instead** | a GPU reset on this chip takes the host down ([akandr/bc250-rocm](https://github.com/akandr/bc250-rocm) "Known issues"); the MastaG README says boot with `amdgpu.gpu_recovery=0` so the machine stays reachable over SSH after a hang |
| `amdgpu.bc250_flush_by_runlist=1` | optional, no effect on Vulkan | a KFD (ROCm) workaround; MastaG carries it opt-in (`docs/ROCM.md`). Vulkan does not use KFD queues |
| `amdgpu.bc250_sdma_fw=navi12` | **drop** | an arieltune kernel patch; the MastaG kernel has no such parameter. Same fix by hand, if ever needed for ROCm, is the firmware copy in [GabriWar/bc250-rocm-working docs/29](https://github.com/GabriWar/bc250-rocm-working/blob/main/docs/29-the-sdma-firmware-is-the-bug.md) |
| `amdgpu.ppfeaturemask`, `amdgpu.noretry=0`, `amdgpu.sched_hw_submission=2`, `iommu=pt amd_iommu=on` | leave as the kernel/board already has them | tuned for the arieltune kernel's SMU patches; nothing documents them as needed by the MastaG kernel. IOMMU off in BIOS is a supported setup |
| `mitigations=off` | your call | throughput trade, not a requirement (the complete guide says the same) |

Limine applies kernel flags from `/etc/default/limine` (CachyOS boot-manager docs, and arieltune's
README Limine paragraph). Edit the `KERNEL_CMDLINE[default]` line **by hand**, appending to what is
there rather than replacing it:

```sh
sudo nano /etc/default/limine
#   KERNEL_CMDLINE[default]="<existing flags> ttm.pages_limit=3588867 ttm.page_pool_size=3588867 amdgpu.gpu_recovery=0"
sudo limine-mkinitcpio
sudo systemctl reboot
```

Then confirm it took:

```sh
cat /proc/cmdline
cat /sys/module/amdgpu/parameters/gpu_recovery   # 0
```

## 3. Install llmtune

Complete guide step 11, unchanged. Its `install.sh` is a bash script, so it runs from fish too.

```sh
git clone https://github.com/thefriendlyhedgehog/llmtune.git
cd llmtune
./install.sh --setup
llmtune doctor
```

`llmtune doctor` checks the PCI id, `amdgpu`, the render node, the Vulkan ICD, the build toolchain and
the models directory. It does not check the kernel or cmdline.

## 4. Build llama.cpp through llmtune, separately from any hand build

```sh
llmtune build install vulkan
```

This installs the commit pinned in `builds.toml` (`86d86ed`, 2026-07-17), the one upstream blessed for
gfx1013 and the one the MTP profiles were benched on. It lives in llmtune's own build directory, so a
hand-built `~/llama.cpp` is untouched and stays usable for `llama-bench` comparisons. Move the pin only
after a doctor + bench smoke: `llmtune build update vulkan --ref <sha>`.

## 5. Serve, then expose

Complete guide steps 11 and 12. Turn auth on before exposing to the LAN:

```sh
llmtune node list
llmtune node load <name>
llmtune node status
llmtune endpoint auth on
llmtune endpoint expose on
llmtune endpoint
```

## Not needed for llmtune: ROCm

llmtune serves through Vulkan only. ROCm on gfx1013 needs a patched kernel module flag, a self-built
gfx1013 rocBLAS and 13 llama.cpp patches, and measures roughly level with Vulkan (prompt 0.96-1.29x,
decode 0.81-1.01x, [akandr/bc250-rocm](https://github.com/akandr/bc250-rocm) README, 2026-10-03), with a
GPU reset that takes the host down. Stay on Vulkan for a serving box.

## Thermals before long serving runs

Sustained llama.cpp load at 1500 MHz reached 93-94 C on akandr's board, after which the governor drops
to 1000 MHz silently (akandr README, "GPU clock policy"). Watch `sensors` and the governor journal on
the first long session, and finish cooling work before leaving the box serving unattended.
