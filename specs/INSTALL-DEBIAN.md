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
- **Debian payload (rootfs payload)**: a prebuilt, compressed archive of a minimal Debian filesystem tree (e.g., a `tar.zst`) that the installer extracts onto the target root partition so it can install Debian **without** networking.

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
This is the “offline Debian installer” approach: we ship a complete minimal Debian root filesystem as a single archive and extract it onto the target disk.

### Artifact
- Example name: `debian-minbase-amd64-bookworm.tar.zst`

### Contents
A bootable minimal Debian system containing:
- `systemd-sysv`
- `linux-image-amd64`
- `initramfs-tools`
- `e2fsprogs`
- `util-linux`
- `iproute2`
- `passwd` (for `chpasswd`)

Notes:
- Firmware packages are excluded initially.
- No network packages are required for first boot DHCP if using systemd-networkd (see below).

### Build-time generation (CI)
In the ISO build pipeline:
1. Run debootstrap to a staging directory.
2. Configure minimal system files.
3. Install required packages into the staging rootfs.
   - Ensure the Debian kernel and initramfs are present under `/boot` (e.g., by installing `linux-image-amd64` and running `update-initramfs` in the chroot during image build).
   - Set a known root password for the MVP (see “Root credentials”).
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

## Root Credentials (MVP)
Set the root password of the installed Debian system to `123456`.

Notes:
- This is insecure and intended only for early bring-up.
- Follow-up work should replace this with a safer approach (first-boot password change, SSH keys, or provisioning).

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

## Installer UX / Output Requirements
- The installer UI output starts at the top-left of the screen.
- The installer prints **exactly one line per step** as it progresses (e.g., "[OK] Partition disk", "[..] Extract Debian payload", "[ERR] bootctl install").
- Avoid centered “welcome” text that blocks/logically interrupts the step log.

## Verification Plan
- Boot ISO in UTM:
  - Confirm disk enumeration output.
  - Confirm partitions created and formatted.
  - Confirm reboot enters Debian without ISO.
  - Confirm DHCP assigns IP after boot (`ip a`).

- Smoke test on real UEFI hardware:
  - Confirm ESP creation.
  - Confirm boot entry works.

## Installer Execution Sequence (MVP)
The installer should run the following steps in order and abort on the first failure:

1. **Enumerate disks** and print a table of candidates + the chosen target disk.
   - If more than one eligible disk exists, abort.
2. **Safety checks** on the chosen target:
    - Confirm it is not currently mounted (no mounted children).
    - Confirm it meets minimum size threshold.
    - Confirm it is writable and non-removable (per current safety filters).
3. **Wipe signatures** (recommended): `wipefs -a <disk>`.
4. **Partition** the disk as GPT with:
    - ESP 512 MiB
    - Root = remainder
5. **Re-read partition table** (e.g., `partprobe` or a short udev settle, depending on available tooling).
6. **Format**:
    - `mkfs.vfat -F 32 <esp>`
    - `mkfs.ext4 -F <root>`
7. **Mount**:
    - Root → `/mnt`
    - ESP → `/mnt/boot/efi`
8. **Install Debian payload** by extracting into `/mnt`.
9. **Set root password** in the target.
   - Example (inside installer environment):
     - `chroot /mnt /bin/sh -lc 'echo "root:123456" | chpasswd'`
9. **Write `/etc/fstab`** in the target using UUIDs.
10. **Configure bootloader** in the target:
      - `chroot /mnt bootctl install`
      - Write loader config and entry that reference the installed kernel/initramfs and `root=UUID=<root-uuid>`.
11. **Configure first-boot DHCP** (systemd-networkd) in the target and enable services.
12. **Sync + unmount** all mounts.
13. **Reboot**.

## Work Breakdown (per repo)
This is the concrete engineering work implied by the spec.

### `installer/` (Rust)
- Disk enumeration via `/sys/block` + safety filters (existing mounted checks included).
   - Abort if more than one eligible disk exists (MVP safety gate).
- Partitioning implementation (choose one):
   - `sfdisk` scripted GPT layout, or
   - `parted` non-interactive.
- Filesystem formatting wrappers and mount orchestration.
- Payload discovery + extraction (see “Payload placement” below).
- Root password set (MVP) via chroot (e.g., `chpasswd`).
- Target configuration writes:
   - `/etc/fstab` using UUIDs
   - systemd-boot loader files
   - systemd-networkd DHCP config + `systemctl enable ...` in chroot
- Error handling:
   - one clear “step name” per stage
   - propagate stderr to console
   - fail closed on any ambiguity (e.g., no eligible disks)

### `installer-kernel/`
- Ensure initramfs contains the required binaries and their runtime deps:
   - partitioning: `sfdisk` or `parted` (+ `partprobe` if used)
   - filesystems: `mkfs.vfat`, `mkfs.ext4`
   - mount utils: `mount`, `umount`
   - payload: `tar` with zstd support (or `tar` + `zstd`)
   - optional: `wipefs`
- Confirm the initramfs does **not** start any DHCP client or otherwise bring up networking.

### `installer-iso/`
- Extend the ISO build so the Debian rootfs payload artifact is embedded into the initramfs and available at runtime at a stable path (recommend: `/payload/debian-rootfs.tar.zst`).
- Keep release build coupling between kernel + installer + payload (same version/revision).

## Payload Placement
For MVP, the payload lives **inside initramfs**.

- Payload is available at a fixed path (recommend: `/payload/debian-rootfs.tar.zst`).
- The installer reads from that path and extracts to `/mnt`.
- No need to mount the ISO filesystem at runtime.

Follow-up option (not MVP): store the payload as a file on the ISO and mount `/dev/sr0` read-only to access it.

## Safety Gates (recommended)
Because this is destructive, the MVP safety gate is:
- Abort if more than one eligible disk is present.

Follow-up options (not required for MVP):
- Require a kernel cmdline flag (e.g., `truthdb.install=1`) before doing any writes.
- Require explicit target disk override (e.g., `truthdb.disk=/dev/sda`) and refuse auto-pick.

## Open Questions
- Target Debian suite and kernel strategy:
  - `bookworm` pinned, or track `stable`?
- Disk selection safety:
  - require an explicit kernel cmdline flag to allow destructive install?

