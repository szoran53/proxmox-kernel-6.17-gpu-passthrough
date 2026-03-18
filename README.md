# Proxmox VE Kernel 6.17 — GPU Passthrough Fork

A fork of the official [Proxmox VE kernel build repo](https://git.proxmox.com/?p=pve-kernel.git)
targeting kernel **6.17.x** with additional patches to fix GPU passthrough reliability on
Proxmox VE 9.x (Debian 13 Trixie).

Like the upstream Proxmox kernel, this build uses the **Ubuntu kernel sources** (not Debian)
as the upstream base. Ubuntu carries important driver and hardware enablement patches that
improve compatibility with newer hardware and PCIe/IOMMU subsystems.

---

## Why This Fork Exists

Proxmox VE 9 ships kernel 6.17, which introduced **AppArmor 5.0.0** — a significant
rewrite of the AppArmor subsystem. This introduced two regressions that break container
operations (LXC, Podman, Docker):

1. **NULL pointer dereference crash** in `__unix_needs_revalidation()` when file
   descriptors are passed via SCM_RIGHTS — causing kernel panics.
2. **Incorrect Unix socket permission classification** for `sendmsg`/`recvmsg` —
   AppArmor classifies socket sends as file reads, causing denials.

Additionally, this fork carries the standard Proxmox GPU passthrough patches and documents
the full system configuration required for reliable NVIDIA/AMD GPU passthrough.

Additionally, this fork documents and carries a patch for a **new issue specific to
NVIDIA Blackwell-generation GPUs** (RTX 5060, 5060 Ti, and likely the broader 50-series):

3. **Firmware 1:1 IOMMU mapping rejection** — Blackwell GPUs set a firmware flag
   requesting a unity IOMMU mapping. When `iommu=pt` is active, the kernel refuses
   to hand the device to VFIO with the error:
   `"Firmware has requested this device have a 1:1 IOMMU mapping, rejecting..."`.
   The patch comments out the `return -EINVAL` in `vfio_pci_core.c` to allow
   passthrough to proceed.

---

## 2. Add row 15 to the Patches Applied table

| 15 | `0015-vfio-pci-bypass-firmware-1to1-iommu-mapping-check.patch` | **Custom:** Bypass Blackwell GPU firmware 1:1 IOMMU mapping rejection in `vfio_pci_core.c`. Required for RTX 5060/5060 Ti passthrough with `iommu=pt`. **Test lab only.** | ✅ |

See [VFIO-1TO1-MAPPING-BYPASS.md](VFIO-1TO1-MAPPING-BYPASS.md) for full technical
explanation, build instructions, and security considerations.

---

## 3. Add to Hardware Tested table

| Passthrough GPU (alt) | NVIDIA GeForce RTX 5060 (GB206) — PCI IDs `10de:2d05`, `10de:22eb` |

---

## 4. Add to Related Links section

* [Proxmox Bugzilla #7374](https://bugzilla.proxmox.com/show_bug.cgi?id=7374) — VFIO 1:1 mapping issue thread

---

## Hardware Tested

| Component | Details |
|-----------|---------|
| CPU | AMD Ryzen 5 9600X (Raphael/Granite Ridge) |
| Motherboard | AMD 600-series chipset |
| Passthrough GPU | NVIDIA GeForce RTX 5060 Ti (GB206) — PCI IDs `10de:2d05`, `10de:22eb` |
| Host display | AMD Radeon Graphics (integrated, Granite Ridge) |
| Proxmox VE | 9.0.3 (Debian 13 Trixie) |
| ZFS | Root-on-ZFS (`rpool/ROOT/pve-1`) |

---

## Patches Applied

All patches are in `patches/kernel/`. The first 12 are the standard Proxmox patches;
patches 13 and 14 are additions in this fork.

| # | File | Purpose | GPU Passthrough Relevant |
|---|------|---------|:---:|
| 01 | `0001-Make-mkcompile_h-accept-an-alternate-timestamp-strin.patch` | Build reproducibility fix | |
| 02 | `0002-wireless-Add-Debian-wireless-regdb-certificates.patch` | Wireless regulatory certs | |
| 03 | `0003-bridge-keep-MAC-of-first-assigned-port.patch` | Bridge MAC stability | |
| 04 | `0004-pci-Enable-overrides-for-missing-ACS-capabilities-4..patch` | Enables `pcie_acs_override` kernel param for IOMMU group splitting | ✅ |
| 05 | `0005-kvm-disable-default-dynamic-halt-polling-growth.patch` | KVM performance | |
| 06 | `0006-net-core-downgrade-unregister_netdevice-refcount-lea.patch` | Networking stability | |
| 07 | `0007-Revert-fortify-Do-not-cast-to-unsigned-char.patch` | Compiler compatibility | |
| 08 | `0008-kvm-xsave-set-mask-out-PKRU-bit-in-xfeatures-if-vCPU.patch` | KVM vCPU correctness | |
| 09 | `0009-allow-opt-in-to-allow-pass-through-on-broken-hardwar.patch` | Adds `intel_iommu=relax_rmrr` — allows passthrough on hardware with non-relaxable RMRRs | ✅ |
| 10 | `0010-revert-memfd-improve-userspace-warnings-for-missing-.patch` | Userspace warning revert | |
| 11 | `0011-apparmor-expect-msg_namelen-0-for-recvmsg-calls.patch` | AppArmor recvmsg fix | |
| 12 | `0012-Revert-UBUNTU-SAUCE-iommu-intel-disable-DMAR-for-SKL.patch` | **Reverts Ubuntu patch that disabled DMAR for SKL/KBL/CML Intel iGPUs** — breaks iGPU passthrough | ✅ |
| 13 | `0013-apparmor-fix-NULL-pointer-dereference-in-aa_file_per.patch` | **Custom:** AppArmor 5.0 NULL ptr dereference crash fix | |
| 14 | `0014-apparmor-fix-unix-socket-sendmsg-classification.patch` | **Custom:** AppArmor 5.0 Unix socket sendmsg permission fix | |

See [APPARMOR-5.0-REGRESSION-FIXES.md](APPARMOR-5.0-REGRESSION-FIXES.md) for a detailed
explanation of patches 13 and 14.

---

## Building the Kernel

### Prerequisites

A Proxmox VE 9.x host (or any Debian 13 Trixie system) with internet access.
The build requires roughly 30–50 GB of disk space and 8+ GB RAM.

### Step 1 — Install base build tools

```bash
apt update
apt install -y git devscripts build-essential
```

### Step 2 — Clone this repo

```bash
cd /root
git clone https://github.com/jaminmc/pve-kernel.git
cd pve-kernel
```

### Step 3 — Initialize submodules

This pulls the Ubuntu kernel source and ZFS-on-Linux. It is large (~2 GB):

```bash
git submodule update --init --recursive
```

> **Note:** The Ubuntu kernel submodule is tracked at a specific commit matching
> the upstream Proxmox build. Do not update submodules unless you intend to
> rebase onto a newer kernel version.

### Step 4 — Create build directory and install build dependencies

```bash
make build-dir-fresh
```

Then install the generated build dependencies (replace `proxmox-kernel-6.17.2`
with the actual directory name if it differs):

```bash
mk-build-deps -ir proxmox-kernel-6.17.2/debian/control
```

### Step 5 — Build

```bash
make deb
```

This will take **1–3 hours** depending on CPU core count. The build uses all
available cores automatically (`--jobs=auto`).

On completion you will find the following `.deb` files in the build directory:

```
proxmox-kernel-6.17.2-1-pve_6.17.2-1_amd64.deb    ← kernel image (install this)
proxmox-headers-6.17.2-1-pve_6.17.2-1_amd64.deb   ← headers (needed for DKMS)
proxmox-kernel-6.17_6.17.2-1_all.deb               ← meta package
proxmox-headers-6.17_6.17.2-1_all.deb              ← headers meta package
linux-tools-6.17_6.17.2-1_amd64.deb                ← perf tools
```

### Step 6 — Install the kernel

```bash
dpkg -i proxmox-kernel-6.17.2-1-pve_6.17.2-1_amd64.deb \
        proxmox-headers-6.17.2-1-pve_6.17.2-1_amd64.deb \
        proxmox-kernel-6.17_6.17.2-1_all.deb \
        proxmox-headers-6.17_6.17.2-1_all.deb
```

### Step 7 — Configure GPU passthrough and reboot

Follow the [GPU Passthrough Setup Guide](GPU-PASSTHROUGH-SETUP.md) **before rebooting**.

Then:

```bash
update-grub
update-initramfs -u -k all
reboot
```

### Step 8 — Set as default boot kernel

After rebooting and verifying the kernel works:

```bash
proxmox-boot-tool kernel pin 6.17.2-1-pve
proxmox-boot-tool refresh
```

---

## GPU Passthrough Setup

See [GPU-PASSTHROUGH-SETUP.md](GPU-PASSTHROUGH-SETUP.md) for the complete
system configuration guide covering:

- GRUB kernel parameters (IOMMU, ACS override, framebuffer disable)
- VFIO module loading order
- GPU blacklisting
- Unsafe interrupts and KVM MSR settings
- Proxmox VM configuration
- Verification and troubleshooting

---

## vendor-reset (AMD GPU users)

If you are passing through an **AMD GPU** (Polaris/Vega/Navi series), install the
`vendor-reset` DKMS module. This fixes the AMD GPU reset bug that prevents
re-initialization after VM shutdown.

```bash
apt install -y dkms linux-headers-$(uname -r)
# or use the proxmox headers package:
# dpkg -i proxmox-headers-6.17.2-1-pve_6.17.2-1_amd64.deb

git clone https://github.com/gnif/vendor-reset.git /opt/vendor-reset
cd /opt/vendor-reset
dkms install .

# Load early at boot
echo 'vendor-reset' >> /etc/modules-load.d/modules.conf
update-initramfs -u -k all
```

> **Note:** `vendor-reset` is **not required for NVIDIA GPUs**. NVIDIA handles
> device reset correctly via FLR (Function Level Reset).

---

## Rebuilding for a New Kernel Version

To track a new upstream Ubuntu/Proxmox kernel version:

```bash
# Update Makefile version variables
# KERNEL_MAJ, KERNEL_MIN, KERNEL_PATCHLEVEL, KREL

# Update the ubuntu-kernel submodule to the new tag
cd submodules/ubuntu-kernel
git fetch origin
git checkout Ubuntu-X.Y.Z-A.B   # target Ubuntu kernel tag
cd ../..

# Re-run build
make clean
make deb
```

Check if any patches need rebasing after a version bump:

```bash
# In the build dir, check if patches apply cleanly
make build-dir-fresh 2>&1 | grep -i "patch\|fail\|reject"
```

---

## Related Links

- [Proxmox VE pve-kernel (upstream)](https://git.proxmox.com/?p=pve-kernel.git)
- [Ubuntu Kernel source mirror](https://git.proxmox.com/?p=mirror_ubuntu-kernels.git)
- [vendor-reset (AMD GPU reset fix)](https://github.com/gnif/vendor-reset)
- [Proxmox GPU Passthrough Wiki](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Proxmox Community Forum — GPU Passthrough](https://forum.proxmox.com/forums/proxmox-ve-configuration.12/)

---

## Credits

- Kernel source and base patches: [Proxmox VE Team](https://www.proxmox.com)
- Ubuntu kernel base: [Canonical/Ubuntu Kernel Team](https://kernel.ubuntu.com)
- AppArmor regression analysis and patches: discovered during Proxmox 9 / kernel 6.17 testing
- PCI ACS override patch: maintained by Proxmox, originally from the community
- RMRR relaxation patch (`0009`): adapted from [kiler129/relax-intel-rmrr](https://github.com/kiler129/relax-intel-rmrr)
