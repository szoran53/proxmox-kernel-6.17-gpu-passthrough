# VFIO 1:1 IOMMU Mapping Bypass Patch

## Overview

This document describes a patch to `drivers/vfio/pci/vfio_pci_core.c` that bypasses
the firmware-requested 1:1 IOMMU mapping enforcement in VFIO. This is required for
passthrough of **NVIDIA Blackwell-generation GPUs** (GB206 / RTX 5060, RTX 5060 Ti,
and likely the broader 50-series family) on Proxmox VE 9.x / kernel 6.17.

> ⚠️ **Test lab / homelab use only.** This patch removes a kernel safety check.
> Do not apply to production systems handling untrusted VMs.

---

## Symptom

When attempting to start a VM with an RTX 5060 or 5060 Ti passed through via VFIO,
the following error appears in `dmesg` and the VM fails to start:

```
vfio-pci 0000:01:00.0: Firmware has requested this device have a 1:1 IOMMU mapping,
rejecting configuring the device without a 1:1 mapping. Contact your platform vendor.
```

QEMU exits immediately:

```
kvm: -device vfio-pci,host=0000:01:00.0,...: vfio 0000:01:00.0: failed to setup
container for group 13: Failed to set group container: Invalid argument
start failed: QEMU exited with code 1
```

This occurs even with a fully correct IOMMU/VFIO configuration — correct GRUB
parameters, correct IOMMU group isolation, and correct VFIO binding.

---

## Root Cause

### Why this happens

NVIDIA Blackwell-generation GPUs (GB206 and related silicon) set a firmware flag
in their PCIe configuration space requesting that the IOMMU driver establish a
**1:1 (unity/identity) mapping** — meaning every guest physical address must map
directly to the same host physical address.

The relevant check in `drivers/vfio/pci/vfio_pci_core.c` (kernel 6.17,
approximately lines 1500–1600) reads:

```c
if (features & VFIO_DEVICE_FEATURE_IOMMU_PASSTHROUGH) {
    if (!iommu_domain_has_feature(domain, IOMMU_DOMAIN_UNITY)) {
        dev_info(&pdev->dev, "Firmware has requested this device have a "
                 "1:1 IOMMU mapping, rejecting configuring the device "
                 "without a 1:1 mapping. Contact your platform vendor.");
        return -EINVAL;
    }
}
```

When `iommu=pt` (passthrough mode) is active in the kernel cmdline, the IOMMU
domain is configured in passthrough mode, which **does not** satisfy the
`IOMMU_DOMAIN_UNITY` feature requirement. The kernel therefore refuses to hand
the device to VFIO.

This is a new enforcement added for Blackwell silicon and does **not** affect
RTX 30/40-series or earlier NVIDIA GPUs, which do not set this firmware flag.

### Why `iommu=pt` conflicts

`iommu=pt` is a performance optimization that tells the IOMMU to use passthrough
mode for DMA — devices access memory directly using host physical addresses without
going through the IOMMU translation tables. This is generally desirable for VFIO
passthrough. However, the unity-mapping check introduced for Blackwell treats
passthrough mode as non-compliant with the firmware request, triggering the rejection.

---

## The Patch

The patch comments out the rejection block in `vfio_pci_core.c`, allowing VFIO
to proceed even when the firmware requests a 1:1 mapping and the IOMMU domain
does not satisfy `IOMMU_DOMAIN_UNITY`.

**File:** `submodules/linux/drivers/vfio/pci/vfio_pci_core.c`

**Location:** ~lines 1500–1600 (search for `IOMMU_DOMAIN_UNITY` or
`Firmware has requested this device have a 1:1 IOMMU mapping`)

**Change:**

```c
/* Before (original): */
if (features & VFIO_DEVICE_FEATURE_IOMMU_PASSTHROUGH) {
    if (!iommu_domain_has_feature(domain, IOMMU_DOMAIN_UNITY)) {
        dev_info(&pdev->dev, "Firmware has requested this device have a "
                 "1:1 IOMMU mapping, rejecting configuring the device "
                 "without a 1:1 mapping. Contact your platform vendor.");
        return -EINVAL;  // ← this return causes the failure
    }
}

/* After (patched): */
if (features & VFIO_DEVICE_FEATURE_IOMMU_PASSTHROUGH) {
    if (!iommu_domain_has_feature(domain, IOMMU_DOMAIN_UNITY)) {
        dev_info(&pdev->dev, "Firmware has requested this device have a "
                 "1:1 IOMMU mapping, rejecting configuring the device "
                 "without a 1:1 mapping. Contact your platform vendor.");
        // return -EINVAL;  // Bypassed: allow passthrough without 1:1 mapping
    }
}
```

The `return -EINVAL` is commented out. The `dev_info` log message is retained
so the condition is still visible in `dmesg`, but execution continues rather
than aborting.

---

## Building the Patched Kernel

### Step 1 — Install build dependencies

```bash
apt install build-essential fakeroot libncurses-dev bison flex \
            libssl-dev libelf-dev dwarves bc git \
            proxmox-headers-6.17.2-1-pve
```

### Step 2 — Clone the repo and check out the branch

```bash
git clone https://github.com/szoran53/proxmox-kernel-6.17-gpu-passthrough.git
cd proxmox-kernel-6.17-gpu-passthrough
git submodule update --init --recursive
```

> The submodule pull is large (~2 GB). This step may take several minutes.

### Step 3 — Apply the patch to vfio_pci_core.c

```bash
nano submodules/linux/drivers/vfio/pci/vfio_pci_core.c
```

Search for `IOMMU_DOMAIN_UNITY` or `Contact your platform vendor`.
Comment out the `return -EINVAL;` line as shown in the patch above. Save the file.

### Step 4 — Build

```bash
make deb
```

Build time is approximately **30–90 minutes** depending on core count.
The build uses all available cores automatically.

On completion, `.deb` files will be in the parent directory:

```
proxmox-kernel-6.17.2-1-pve_6.17.2-1_amd64.deb
proxmox-headers-6.17.2-1-pve_6.17.2-1_amd64.deb
```

### Step 5 — Install the kernel

```bash
dpkg -i proxmox-kernel-6.17.2-1-pve_6.17.2-1_amd64.deb \
        proxmox-headers-6.17.2-1-pve_6.17.2-1_amd64.deb
```

### Step 6 — Pin and boot

```bash
proxmox-boot-tool kernel add $(uname -r | sed 's/-pve//')-pve
proxmox-boot-tool kernel pin 6.17.2-1-pve
update-grub
reboot
```

### Step 7 — Verify

After reboot, the `dmesg` 1:1 mapping message will still appear (the `dev_info`
log line is retained) but the VM should now start successfully:

```bash
dmesg | grep "1:1 IOMMU"
# Message still present — but execution continues past it

qm start <VMID>
# Should no longer fail with "Invalid argument"
```

---

## Additional VM Configuration for RTX 5060 / 5060 Ti

Beyond the kernel patch, the VM config requires specific settings for Blackwell GPUs.

### Pass through both GPU functions

Both the video and audio functions must be in the VM config. They share IOMMU group
13 on most Raphael/Granite Ridge systems:

```ini
hostpci0: 0000:01:00.0,pcie=1,rombar=0
hostpci1: 0000:01:00.1,pcie=1
```

Note `rombar=0` on the GPU function — Blackwell cards have a vBIOS ROM BAR that
can cause container setup failures if left enabled.

### Removing `iommu=pt` as an alternative (no kernel build required)

Before resorting to the kernel patch, try removing `iommu=pt` from
`GRUB_CMDLINE_LINUX_DEFAULT`. This may satisfy the firmware 1:1 mapping
requirement without any source modification:

```
# Change this:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=..."

# To this:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on pcie_acs_override=..."
```

Then run `update-grub` and reboot. If passthrough works without `iommu=pt`,
the kernel patch is not needed. The tradeoff is a minor DMA performance overhead
since all devices go through IOMMU translation tables.

If removing `iommu=pt` alone is insufficient (VM still fails), then the kernel
patch is required.

---

## Affected Hardware

This issue is confirmed on:

| GPU | PCI ID | Architecture |
|-----|--------|--------------|
| GeForce RTX 5060 | `10de:2d05` | GB206 (Blackwell) |
| GeForce RTX 5060 Ti | `10de:2d05` | GB206 (Blackwell) |

Likely affected (same firmware flag expected):

| GPU | Architecture |
|-----|--------------|
| GeForce RTX 5070 | GB205 (Blackwell) |
| GeForce RTX 5070 Ti | GB203 (Blackwell) |
| GeForce RTX 5080 | GB203 (Blackwell) |
| GeForce RTX 5090 | GB202 (Blackwell) |

Not affected (pre-Blackwell — firmware flag not set):

* RTX 40-series (Ada Lovelace)
* RTX 30-series (Ampere)
* RTX 20-series (Turing)

---

## Security Considerations

This patch bypasses a firmware safety request. The implications for a homelab
single-tenant hypervisor are minimal — you own both the host and guest — but
they are worth understanding:

- The 1:1 mapping request exists to ensure DMA coherence and prevent certain
  classes of memory access violations between the device and the IOMMU.
- In a multi-tenant environment, bypassing this check could weaken isolation
  between VMs sharing the same IOMMU domain.
- For a single-GPU single-VM passthrough configuration (the typical homelab
  scenario), the practical risk is negligible.

**Do not deploy this patch on shared or production hypervisors.**

---

## Relationship to Other Patches in This Fork

This patch is **independent** of the other patches in this fork:

| Patch | Scope | Related to this issue? |
|-------|-------|------------------------|
| `0004` pcie_acs_override | IOMMU group splitting | ❌ No — your groups are already clean |
| `0009` relax_rmrr | Intel RMRR relaxation | ❌ No — AMD platform, no RMRRs involved |
| `0012` Revert Ubuntu DMAR disable | Intel SKL/KBL/CML iGPU DMAR | ❌ No — AMD platform |
| `0013` AppArmor NULL ptr fix | LXC/container stability | ❌ No — separate issue |
| `0014` AppArmor sendmsg fix | LXC/container permissions | ❌ No — separate issue |
| **this patch** vfio_pci_core.c | Blackwell 1:1 mapping bypass | ✅ This is the fix |

The ACS override (`0004`) and RMRR relaxation (`0009`) patches documented
elsewhere in this repo solve IOMMU group isolation and Intel-specific RMRR
constraint problems respectively. If you are on AMD with a RTX 5060/5060 Ti and
your IOMMU groups are already clean (GPU isolated in its own group), those patches
are not what you need — this one is.

---

## References

- Proxmox Bugzilla #7374
- `drivers/vfio/pci/vfio_pci_core.c` — upstream kernel source
- NVIDIA GB206 PCI IDs: `10de:2d05` (video), `10de:22eb` (audio)
- [GPU-PASSTHROUGH-SETUP.md](GPU-PASSTHROUGH-SETUP.md) — full system configuration guide
