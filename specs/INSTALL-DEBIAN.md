# INSTALL-DEBIAN.md

> Note: This lives in the org `.github` repo by request, but this repo’s README indicates product/architecture docs usually belong in the relevant product repository (e.g., `installer/` or `installer-iso/`). If we want to align with that, we can relocate this spec later.

## Goal
Build a TruthDB installer ISO that, when booted on a UEFI machine, automatically installs a minimal Debian system onto the first eligible disk and reboots into that Debian system.

Constraints:
- UEFI-only (no legacy BIOS support).
- No network is required during installation.
- After Debian boots, the installed system should obtain an IP via DHCP automatically.
- Installation is unattended (no prompts) in the first iteration.

Explicit networking policy:
- The installer environment must not attempt to bring up networking (no DHCP, no downloads).

## Definitions
- **Installer environment**: the initramfs userspace launched by the ISO (BusyBox + `truthdb-installer`).
- **Target disk**: the block device selected for installation (e.g., `/dev/vda`, `/dev/sda`, `/dev/nvme0n1`).
- **ESP**: EFI System Partition mounted at `/boot/efi` in the installed system.

## Non-Goals (initially)
- Disk selection UI.
- Encryption (LUKS), LVM, RAID.
- Preserving existing data.
- Dual boot.

## End State (Acceptance Criteria)
After booting the ISO:
1. Installer prints the disks it detected and the chosen target disk.
2. Installer partitions and formats the target disk:
   - GPT partition table.
   - Partition 1: ESP (FAT32), 512 MiB.
   - Partition 2: Debian root (EXT4), remainder.
3. Installer installs an offline minimal Debian root filesystem to the root partition.
4. Installer sets up UEFI boot with `systemd-boot` so the machine boots from the target disk without the ISO.
5. On first boot of the installed Debian system:
   - The system reaches `multi-user.target`.
   - An Ethernet interface uses DHCP automatically.

## High-Level Approach
We will **embed a prebuilt minimal Debian root filesystem payload** inside the ISO (or initramfs) and have the installer write it to disk.

This avoids needing debootstrap, apt, mirrors, DNS, or DHCP during installation.

## Repository Responsibilities
This work spans multiple repos:

1. **installer** (Rust)
   - Add logic to detect disks, partition, format, mount, extract rootfs payload, configure boot.

2. **installer-iso**
   - Update ISO build workflow to embed the Debian rootfs payload artifact into the initramfs.
   - Ensure release builds use version-matched kernel+installer (already being enforced).

3. **installer-kernel**
   - Ensure kernel/initramfs includes the necessary tooling for partitioning, formatting, mounting, and extraction.

## Disk Detection and Selection
### Detection source
Use `/sys/block` to enumerate devices.

### Candidate filtering
Skip the following device classes:
- `loop*`, `ram*`, `sr*`, `fd*`, `dm-*`, `md*`.

Skip partitions; only accept whole disks (examples):
- Accept: `sda`, `vda`, `nvme0n1`.
- Reject: `sda1`, `vda2`, `nvme0n1p1`.

### Safety filters
Exclude disks that are:
- read-only (`/sys/block/<dev>/ro == 1`)
- removable (`/sys/block/<dev>/removable == 1`) (first iteration)
- smaller than a minimum size threshold (recommend: 8 GiB)

### Ordering
Sort candidate disks lexicographically by kernel name and choose the first.

### Logging
Print for each disk:
- `/dev/<name>`
- size in GiB
- `removable`, `ro`
- model/vendor when available

## Partitioning Scheme (UEFI)
Target: GPT with 2 partitions.

- **Partition 1 (ESP)**
  - Size: 512 MiB
  - Type: EFI System Partition
  - Filesystem: FAT32
  - Mount point: `/boot/efi`

- **Partition 2 (root)**
  - Size: remaining space
  - Filesystem: EXT4
  - Mount point: `/`

Notes:
- NVMe partition naming uses `p1`, `p2`.
- SATA/virtio naming uses `1`, `2`.

## Formatting
- Format ESP: `mkfs.vfat -F 32 <esp>`
- Format root: `mkfs.ext4 -F <root>`

Optional (recommended): wipe old signatures before partitioning:
- `wipefs -a <disk>`

## Mount Layout During Install
Mount points in installer environment:
- `/mnt` → root partition
- `/mnt/boot/efi` → ESP

## Offline Debian Payload
### Artifact
- `debian-minbase-amd64-bookworm.tar.zst`

### Contents
A bootable minimal Debian system containing:
- `systemd-sysv`
- `linux-image-amd64`
- `initramfs-tools`
- `e2fsprogs`
- `util-linux`
- `iproute2`

Notes:
- Firmware packages are excluded initially.
- No network packages are required for first boot DHCP if using systemd-networkd (see below).

### Build-time generation (CI)
In the ISO build pipeline:
1. Run debootstrap to a staging directory.
2. Configure minimal system files.
3. Install required packages into the staging rootfs.
   - Ensure the Debian kernel and initramfs are present under `/boot` (e.g., by installing `linux-image-amd64` and running `update-initramfs` in the chroot during image build).
4. Clean apt caches.
5. Pack as a tarball with numeric ownership.

### Runtime installation
- Extract payload to `/mnt` with preservation of permissions, owners, symlinks:
  - `tar --zstd -xpf <payload> -C /mnt`

## Bootloader Setup (systemd-boot)
### Why systemd-boot
UEFI-only with minimal moving parts.

### Steps (installed system)
Within `chroot /mnt`:
1. Install systemd-boot into the ESP:
   - `bootctl install`
2. Create loader config:
   - `/boot/loader/loader.conf`
3. Create a boot entry:
   - `/boot/loader/entries/debian.conf`

### Boot entry contents
- Use `root=UUID=<root-uuid> ro`.
- Point to the installed Debian kernel and initramfs paths under `/boot`.

## fstab
Write `/mnt/etc/fstab` entries using UUIDs:
- Root ext4 mounted at `/`.
- ESP vfat mounted at `/boot/efi`.

## First Boot DHCP
Configure systemd-networkd to bring up DHCP automatically.

### Files
Create `/mnt/etc/systemd/network/20-dhcp.network`:
- Match common wired interface patterns (e.g., `en*`, `eth*`).
- Enable DHCP.

Enable required services in the target:
- `systemd-networkd.service`
- `systemd-resolved.service`

Configure `/etc/resolv.conf` to use systemd-resolved stub.

## Installer Runtime Dependencies
The initramfs environment must include:
- Partitioning tool: `sfdisk` or `parted`
- Formatting tools: `mkfs.vfat`, `mkfs.ext4`
- Mount tooling: `mount`, `umount`
- Extraction tooling: `tar` with zstd support (or separate `zstd` + tar)
- `wipefs` (optional)

## Error Handling Requirements
Installer must:
- Abort if target disk appears mounted.
- Abort if any command fails; print the failing step.
- Leave clear logs on console for post-mortem.

## Verification Plan
- Boot ISO in UTM:
  - Confirm disk enumeration output.
  - Confirm partitions created and formatted.
  - Confirm reboot enters Debian without ISO.
  - Confirm DHCP assigns IP after boot (`ip a`).

- Smoke test on real UEFI hardware:
  - Confirm ESP creation.
  - Confirm boot entry works.

## Open Questions
- Target Debian suite and kernel strategy:
  - `bookworm` pinned, or track `stable`?
- Root credentials:
  - preset root password, create default user, or require later provisioning?
- Disk selection safety:
  - require an explicit kernel cmdline flag to allow destructive install?
