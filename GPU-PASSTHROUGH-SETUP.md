# GPU Passthrough Setup Guide for Proxmox VE 9.x

Complete system configuration for reliable PCI GPU passthrough on Proxmox VE 9.x
running kernel 6.17.x.

**Tested hardware:**
- CPU: AMD Ryzen 5 9600X (Raphael/Granite Ridge — `amd_iommu`)
- GPU: NVIDIA GeForce RTX 5060 Ti (PCI IDs: `10de:2d05`, `10de:22eb`)
- Host display: AMD Radeon integrated graphics (kept on host)

> For Intel systems, replace `amd_iommu=on` with `intel_iommu=on` in the GRUB
> parameters below. The rest of the guide applies equally.

---

## Step 1 — BIOS/UEFI Settings

Before anything else, enable in your motherboard BIOS:

- **IOMMU** (also called AMD-Vi on AMD, VT-d on Intel)
- **SR-IOV** (if present — improves IOMMU group isolation)
- **Above 4G Decoding** — required for most modern discrete GPUs
- **Resizable BAR / Smart Access Memory** — enable if available

Disable:

- **CSM / Legacy Boot** — use UEFI mode only
- **Fast Boot** — can interfere with PCIe device initialization

---

## Step 2 — GRUB Kernel Parameters

Edit `/etc/default/grub`:

```bash
nano /etc/default/grub
```

Set `GRUB_CMDLINE_LINUX_DEFAULT`:

```ini
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset video=vesafb:off,efifb:off"
```

**Parameter breakdown:**

| Parameter | Purpose |
|-----------|---------|
| `amd_iommu=on` | Enable AMD IOMMU (use `intel_iommu=on` for Intel) |
| `iommu=pt` | Passthrough mode — only map devices actually being passed through (better performance, avoids IOMMU mapping conflicts) |
| `pcie_acs_override=downstream,multifunction` | Override missing ACS capabilities to allow IOMMU group splitting. Enabled by kernel patch `0004`. **Note:** reduces IOMMU isolation — acceptable for home lab use |
| `nofb` | Disable framebuffer at kernel level |
| `nomodeset` | Prevent kernel from setting video mode (keep GPU uninitialized for passthrough) |
| `video=vesafb:off,efifb:off` | Disable legacy framebuffer drivers that can grab the GPU before VFIO |

**Intel-specific addition** (if using Intel platform with non-relaxable RMRRs):

```ini
GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on,relax_rmrr ..."
```

The `relax_rmrr` option is enabled by kernel patch `0009`.

Apply changes:

```bash
update-grub
```

---

## Step 3 — Blacklist Host GPU Drivers

Prevent the host kernel from loading NVIDIA/Nouveau drivers for the passthrough GPU.

Create `/etc/modprobe.d/blacklist-nvidia.conf`:

```bash
cat > /etc/modprobe.d/blacklist-nvidia.conf << 'EOF'
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
EOF
```

> If you are passing through an **AMD GPU** instead, blacklist the AMD driver:
> ```
> blacklist amdgpu
> blacklist radeon
> ```
> But only blacklist the GPU being passed through — if you have an AMD iGPU for
> the host display, do NOT blacklist amdgpu globally. Instead bind only the
> discrete GPU PCI IDs to vfio-pci (Step 4).

---

## Step 4 — Bind the GPU to VFIO-PCI

Find your GPU's PCI IDs:

```bash
lspci -nn | grep -i "vga\|audio\|nvidia\|amd"
```

Example output (RTX 5060 Ti):
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GB206 [GeForce RTX 5060 Ti] [10de:2d05]
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22eb]
```

Create `/etc/modprobe.d/vfio.conf` with **both** the GPU and its audio function:

```bash
cat > /etc/modprobe.d/vfio.conf << 'EOF'
options vfio-pci ids=10de:2d05,10de:22eb disable_vga=1
EOF
```

Replace `10de:2d05,10de:22eb` with your actual PCI IDs.

> `disable_vga=1` prevents vfio-pci from handling legacy VGA I/O ranges,
> which can conflict with the host display adapter.

---

## Step 5 — VFIO Module Loading

Ensure VFIO modules load early in the boot process (before any GPU driver can claim the device).

Edit `/etc/modules-load.d/modules.conf`:

```bash
cat > /etc/modules-load.d/modules.conf << 'EOF'
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
EOF
```

---

## Step 6 — VFIO IOMMU and KVM Options

**Allow unsafe interrupts** (needed if your hardware does not support interrupt remapping):

```bash
cat > /etc/modprobe.d/iommu_unsafe_interrupts.conf << 'EOF'
options vfio_iommu_type1 allow_unsafe_interrupts=1
EOF
```

**KVM MSR passthrough** (prevents Windows VMs crashing on unknown MSRs):

```bash
cat > /etc/modprobe.d/kvm.conf << 'EOF'
options kvm ignore_msrs=1
EOF
```

---

## Step 7 — Update initramfs

Regenerate initramfs so VFIO modules are baked in early:

```bash
update-initramfs -u -k all
```

---

## Step 8 — Reboot and Verify

```bash
reboot
```

After reboot, verify IOMMU is active:

```bash
dmesg | grep -i iommu | head -20
```

You should see lines like:
```
AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40
pci 0000:01:00.0: Adding to iommu group 15
```

Verify the GPU is bound to vfio-pci (not nvidia/nouveau):

```bash
lspci -nnk -d 10de:
```

Expected output:
```
01:00.0 VGA compatible controller [10de:2d05]:
        Kernel driver in use: vfio-pci
        Kernel modules: nouveau
01:00.1 Audio device [10de:22eb]:
        Kernel driver in use: vfio-pci
```

Check IOMMU groups for the passthrough GPU:

```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'Group %s: ' "$n"
    lspci -nns "${d##*/}"
done | grep -A5 "10de:"
```

Ideally the GPU and its audio function should be the **only** devices in their IOMMU group.
If other devices share the group, `pcie_acs_override` in the GRUB params (Step 2) should
have split them — if not, check that the kernel patch `0004` was applied.

---

## Step 9 — Proxmox VM Configuration

In the Proxmox web UI or via CLI, configure the VM:

### Machine type
Set to **q35** (required for PCIe passthrough):
```
Machine: q35
```

### BIOS
Set to **OVMF (UEFI)**:
```
BIOS: OVMF (UEFI)
```
Add an EFI disk when prompted.

### PCI Device
Add the GPU via **Hardware → Add → PCI Device**:

| Setting | Value |
|---------|-------|
| Device | `01:00` (your GPU's address) |
| All Functions | ✅ checked |
| ROM-Bar | ✅ checked |
| PCI-Express | ✅ checked |
| Primary GPU | ✅ if this is the only display output |

### CPU
```
CPU type: host
```

Using `host` CPU type exposes the actual CPU flags to the VM, which is important
for NVIDIA driver compatibility and performance.

### Display
If using the GPU as the primary display (no VNC after boot):
```
Display: none
```

Or keep `VirtIO` / `std` for a secondary console during setup.

### Example VM config (`/etc/pve/qemu-server/<VMID>.conf`)

```ini
machine: pc-q35-9.2
bios: ovmf
cpu: host
cores: 4
sockets: 1
memory: 16384
balloon: 0
efidisk0: local-lvm:vm-100-disk-0,efitype=4m,pre-enrolled-keys=1,size=4M
hostpci0: 0000:01:00,allFunctions=1,pcie=1,rombar=1,x-vga=1
vga: none
```

---

## Step 10 — Windows VM Extras (NVIDIA)

For Windows VMs with NVIDIA passthrough:

### Hyper-V enlightenments (improves performance and stability)

Add to VM config:
```ini
cpu: host,hidden=1,flags=+pcid
args: -cpu host,hv_vendor_id=proxmox,kvm=off
```

The `kvm=off` and `hv_vendor_id` prevent NVIDIA drivers from detecting the
hypervisor and refusing to load (NVIDIA's "Error 43" on older drivers).

> Modern NVIDIA drivers (537+) no longer enforce this, but setting it does
> not hurt and maintains compatibility.

### VirtIO drivers

Install [VirtIO drivers for Windows](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/)
for disk and network performance.

---

## Troubleshooting

### GPU not binding to vfio-pci

Check that the GPU PCI IDs in `/etc/modprobe.d/vfio.conf` match exactly:

```bash
lspci -nn | grep -i nvidia   # note the [xxxx:yyyy] at the end
```

Regenerate initramfs and reboot:
```bash
update-initramfs -u -k all && reboot
```

### IOMMU groups still contain other devices

Verify `pcie_acs_override=downstream,multifunction` is in your kernel cmdline:
```bash
cat /proc/cmdline
```

If still grouped with other devices, check the BIOS ACS settings or try
`pcie_acs_override=id:xxxx:yyyy` to target specific devices.

### VM crashes or GPU resets fail (AMD GPU only)

Install and load `vendor-reset` early — see the README.

### dmesg shows "DMAR: DRHD: handling fault status reg"

This is an IOMMU fault. Common causes:
- GPU ROM issues: try `rombar=0` in the VM PCI device config
- Missing `iommu=pt` in kernel cmdline
- Device isolation issue: check IOMMU groups

### NVIDIA "Code 43" error in Windows (older drivers only)

Add to VM config:
```ini
args: -cpu host,hv_vendor_id=NvidiaFGPA,kvm=off
```

### AppArmor crashes with containers after upgrading to kernel 6.17

This is fixed by patches 0013 and 0014 in this kernel fork.
Verify the patches are present:
```bash
uname -r   # should show 6.17.x-pve
dmesg | grep -i "null pointer"   # should be empty
```

### Checking VFIO is working end-to-end

```bash
# List VFIO devices
ls /dev/vfio/

# Check which VFIO group your GPU is in
readlink /sys/bus/pci/devices/0000:01:00.0/iommu_group
# e.g. → ../../../../kernel/iommu_groups/15

# The group number should appear in /dev/vfio/
ls /dev/vfio/15
```

---

## File Reference

| File | Purpose |
|------|---------|
| `/etc/default/grub` | GRUB boot parameters (IOMMU, nomodeset) |
| `/etc/modprobe.d/vfio.conf` | Bind GPU PCI IDs to vfio-pci driver |
| `/etc/modprobe.d/blacklist-nvidia.conf` | Prevent host from loading NVIDIA drivers |
| `/etc/modprobe.d/iommu_unsafe_interrupts.conf` | Allow unsafe interrupts for VFIO |
| `/etc/modprobe.d/kvm.conf` | KVM MSR ignore setting |
| `/etc/modules-load.d/modules.conf` | Load VFIO modules at early boot |
