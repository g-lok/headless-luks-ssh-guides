# ZimaBoard 2: Headless Arch Linux with LUKS-Encrypted Root and SSH Unlock

## Workstation-Based Chroot Build → NVMe Install

> **Date**: April 2026
> **Target**: ZimaBoard 2 1664 (Intel N150, 16GB LPDDR5x, 64GB eMMC)
> **NVMe**: Samsung 990 Pro (2280 M-key) via PCIe-to-M.2 adapter card
> **NVMe on host**: `/dev/sdd` (via USB enclosure)
> **Host**: Any x86_64 Linux workstation (Arch, Kali/Debian, Fedora, etc.)
> **Status**: Tested — first boot successful on external SSD (April 2026). NVMe install pending.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
   - [2a. Non-Arch Host Setup (Kali/Debian/Ubuntu)](#2a-non-arch-host-setup-kalidebianubuntu)
3. [Hardware Notes](#3-hardware-notes)
4. [Prepare the NVMe on Your Workstation](#4-prepare-the-nvme-on-your-workstation)
5. [Bootstrap Arch via Chroot](#5-bootstrap-arch-via-chroot)
6. [Configure the System Inside Chroot](#6-configure-the-system-inside-chroot)
7. [Headless SSH LUKS Unlock (tinyssh)](#7-headless-ssh-luks-unlock-tinyssh)
8. [Build UKI and Install Bootloader](#8-build-uki-and-install-bootloader)
9. [Exit Chroot and Clean Up](#9-exit-chroot-and-clean-up)
10. [First Boot on ZimaBoard 2](#10-first-boot-on-zimaboard-2)
11. [Post-Boot Configuration](#11-post-boot-configuration)
12. [Gotchas and Lessons Learned](#12-gotchas-and-lessons-learned)
13. [SSH Config for Key Separation](#13-ssh-config-for-key-separation)
14. [Quick Reference](#14-quick-reference)
15. [Resources](#15-resources)
16. [Appendix A: Shrink, Image, Restore, Expand](#appendix-a-shrink-image-restore-expand)

---

## 1. Overview

This guide builds a complete, encrypted, headless Arch Linux installation on an NVMe SSD **entirely from your workstation via chroot** — no need to boot the Arch ISO on the ZimaBoard itself. The NVMe is then installed into the ZimaBoard and boots directly.

> **Non-Arch hosts (Kali, Debian, Ubuntu, Fedora, etc.):** This guide works from any x86_64 Linux workstation. If you're not on Arch, see [Section 2a](#2a-non-arch-host-setup-kalidebianubuntu) for one-time setup of the Arch bootstrap tools. Once those are in place, all instructions are identical.

The process:

1. Connect Samsung 990 Pro to workstation (M.2 slot or USB enclosure)
2. Partition, encrypt with LUKS2, format
3. `pacstrap` a full Arch system into the encrypted root
4. Configure mkinitcpio with systemd hooks, tinyssh SSH, networking
5. Build a Unified Kernel Image (UKI) with systemd-boot
6. Install NVMe into ZimaBoard 2 via PCIe adapter
7. Change BIOS boot order, first boot
8. SSH in to unlock LUKS, system boots headless

---

## 2. Prerequisites

### Hardware

- ZimaBoard 2 (any SKU)
- Samsung 990 Pro NVMe SSD (2280 M-key) or any external USB SSD.
- **PCIe-to-M.2 NVMe adapter card** — the ZimaBoard's PCIe slot is a standard desktop-style PCIe x4 slot, NOT an M.2 slot. You need an adapter. Zima sells one, or any generic PCIe x4 to M.2 M-key adapter works.
- A way to connect the NVMe to your workstation: M.2 slot on the motherboard, or USB-to-NVMe enclosure
- Mini DisplayPort to HDMI adapter + monitor + USB keyboard — **required for first boot** to change BIOS boot order. No serial console or IPMI available on ZimaBoard 2.
- Ethernet cable (connected to same network as workstation)

### Software (on workstation)

**If your host is Arch Linux:**

```bash
sudo pacman -S arch-install-scripts dosfstools cryptsetup btrfs-progs
```

**If your host is Kali/Debian/Ubuntu:**

```bash
sudo apt install gdisk dosfstools cryptsetup btrfs-progs curl wget \
  pacman-package-manager archlinux-keyring arch-install-scripts
```

The last three packages provide `pacman`, `pacstrap`, `genfstab`, and `arch-chroot`. See [Section 2a](#2a-non-arch-host-setup-kalidebianubuntu) for keyring initialization.

### An SSH key pair

```bash
# If you don't have one:
ssh-keygen -t ed25519 -C "you@workstation"
```

---

## 2a. Non-Arch Host Setup (Kali/Debian/Ubuntu)

> **Skip this section entirely if your workstation runs Arch Linux.**

Building an Arch system via chroot requires three Arch-specific tools: `pacstrap` (bootstraps packages into a target root), `genfstab` (generates `/etc/fstab` from mounted filesystems), and `arch-chroot` (enters the chroot with proper bind mounts).

On Kali/Debian/Ubuntu, all three are available via `apt`:

### Install via apt

```bash
sudo apt update
sudo apt install pacman-package-manager archlinux-keyring arch-install-scripts
```

### Initialize the Arch keyring

```bash
sudo rm -rf /etc/pacman.d/gnupg
sudo pacman-key --init
sudo pacman-key --populate archlinux
```

### Configure pacman mirrorlist

```bash
sudo mkdir -p /etc/pacman.d

cat <<'EOF' | sudo tee /etc/pacman.d/mirrorlist
Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch
Server = https://mirror.rackspace.com/archlinux/$repo/os/$arch
Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch
EOF
```

### Verify

```bash
pacman --version
which pacstrap genfstab arch-chroot
```

All three should be in PATH. From this point, all subsequent sections apply without modification.

> **Keyring failures are the most common problem** on non-Arch hosts. If you get GPG errors during `pacstrap`, the keyring isn't properly initialized. Re-run the keyring steps above. See also [Section 12](#12-gotchas-and-lessons-learned).

### Alternative: Boot the Arch ISO

If apt packages are unavailable or broken on your distro:

1. Download the [Arch Linux ISO](https://archlinux.org/download/)
2. Write it to a USB stick: `sudo dd if=archlinux-*.iso of=/dev/sdX bs=4M status=progress`
3. Boot your workstation from the USB stick
4. The live Arch environment has all tools pre-installed
5. Connect the NVMe via USB enclosure, follow sections 4-9
6. Reboot back into your normal OS when done

---

## 3. Hardware Notes

### ZimaBoard 2 specs (relevant)

| Property | Value                                                          |
| -------- | -------------------------------------------------------------- |
| CPU      | Intel N150 (Twin Lake), 4× E-cores, 3.6 GHz boost              |
| RAM      | 16 GB LPDDR5x 4800MHz (soldered)                               |
| eMMC     | 64 GB (ships with ZimaOS — leave it, boot from NVMe)           |
| PCIe     | 3.0 x4 physical slot, **x2 electrical lanes** (~2 GB/s)        |
| Network  | 2× Intel i226V 2.5GbE (driver: `igc`)                          |
| Video    | Mini DisplayPort 1.4                                           |
| USB      | 2× USB 3.1                                                     |
| TPM      | Intel PTT (fTPM 2.0) — **must be enabled in BIOS**             |
| BIOS     | AMI Aptio (UEFI). **Delete** = BIOS setup, **F11** = boot menu |
| Power    | No power button — powers on when adapter is plugged in         |

### Samsung 990 Pro notes

- PCIe 4.0 x4 drive, but will run at PCIe 3.0 x2 in the ZimaBoard (~2 GB/s). Still vastly faster than eMMC.
- Standard 2280 M-key form factor
- Well-supported in Linux, no special firmware needed
- `linux-firmware` package covers it

### PCIe adapter

- The ZimaBoard's PCIe slot accepts standard low-profile or full-height PCIe cards
- Zima's official adapter supports 2230/2242/2260/2280 M-key NVMe
- Any generic PCIe x4 to M.2 NVMe adapter will work
- **Dual NVMe adapters**: only one slot works (N150 doesn't support PCIe bifurcation)

### Device naming gotcha

- Connected to workstation via USB enclosure: `/dev/sdX` (e.g., `/dev/sdb`)
- Connected to workstation via M.2 slot: `/dev/nvme0n1` or `/dev/nvme1n1`
- Installed in ZimaBoard natively: `/dev/nvme0n1`

**All references must use UUIDs.** Device names WILL change between your workstation and the ZimaBoard.

---

## 4. Prepare the NVMe on Your Workstation

### Identify the disk

Connect the 990 Pro to your workstation and identify it:

```bash
lsblk
```

Set a variable for convenience (replace with your actual device):

```bash
# If via USB enclosure:
DISK=/dev/sdb
# If via native M.2 slot:
DISK=/dev/nvme1n1
```

> **Triple-check you have the right disk.** The next command wipes it.

### Partition

```bash
# Wipe existing partition tables
sudo sgdisk --zap-all $DISK

# Create GPT with two partitions:
#   1: 1 GiB EFI System Partition (UKIs are large — need room for default + fallback)
#   2: Remainder as Linux root (will be LUKS-encrypted)
sudo sgdisk --clear \
  --new=1:0:+1G   --typecode=1:EF00 --change-name=1:"EFI" \
  --new=2:0:0      --typecode=2:8304 --change-name=2:"cryptroot" \
  $DISK

# Verify
sudo sgdisk -p $DISK
```

### Encrypt root partition

```bash
# Determine partition device names
# USB enclosure: ${DISK}2 (e.g., /dev/sdb2)
# Native NVMe:   ${DISK}p2 (e.g., /dev/nvme1n1p2)
# Adjust accordingly:
PART2=${DISK}2      # or ${DISK}p2 for NVMe

sudo cryptsetup luksFormat --type luks2 $PART2
# Enter and confirm passphrase
```

> **PBKDF note**: LUKS2 defaults to Argon2id with ~1 GiB memory. The ZimaBoard has 16 GB RAM, so this is fine. If you're worried about key derivation time on the N150 vs your workstation, you can benchmark: `cryptsetup benchmark`. For a consistent experience, you can set explicit parameters:
>
> ```bash
> sudo cryptsetup luksFormat --type luks2 --pbkdf argon2id \
>   --iter-time 3000 --pbkdf-memory 1048576 $PART2
> ```

### Open LUKS and format filesystems

```bash
sudo cryptsetup open $PART2 root
# Enter passphrase

# Root filesystem (btrfs with zstd compression)
sudo mkfs.btrfs -L archroot /dev/mapper/root

# EFI System Partition
PART1=${DISK}1      # or ${DISK}p1 for NVMe
sudo mkfs.fat -F32 -n EFI $PART1
```

### Create btrfs subvolumes

Mount the raw btrfs volume first, create subvolumes, then remount with subvolume targets:

```bash
sudo mount /dev/mapper/root /mnt

# Create subvolume layout
sudo btrfs subvolume create /mnt/@
sudo btrfs subvolume create /mnt/@home
sudo btrfs subvolume create /mnt/@snapshots
sudo btrfs subvolume create /mnt/@var_log
sudo btrfs subvolume create /mnt/@var_cache

# Unmount the raw volume
sudo umount /mnt
```

**Subvolume layout explained:**

| Subvolume    | Mountpoint    | Purpose                                     |
| ------------ | ------------- | ------------------------------------------- |
| `@`          | `/`           | Root filesystem                             |
| `@home`      | `/home`       | User data (separate snapshot policy)        |
| `@snapshots` | `/.snapshots` | Snapshot storage                            |
| `@var_log`   | `/var/log`    | Logs (exclude from root snapshots)          |
| `@var_cache` | `/var/cache`  | Package cache (exclude from root snapshots) |

> **Why subvolumes?** When you snapshot `@` before a system update, you don't want gigabytes of package cache or logs in the snapshot. Separate subvolumes let you snapshot root cleanly. `@home` gets its own snapshot schedule.

### Note BOTH UUIDs (CRITICAL — #1 source of boot failures)

There are **two different UUIDs** involved. Confusing them causes the exact boot failure shown in the error screenshot (systemd times out looking for a device that doesn't exist).

```bash
# 1. LUKS CONTAINER UUID — the encrypted partition itself
sudo blkid $PART2
# Example: UUID="824f87fe-..." TYPE="crypto_LUKS"
# Used in: /etc/crypttab.initramfs, kernel cmdline (rd.luks.name=)

# 2. FILESYSTEM UUID — the btrfs inside the unlocked LUKS
sudo blkid /dev/mapper/root
# Example: UUID="2cbc57e5-..." TYPE="btrfs"
# Used in: /etc/fstab (all subvolume mounts)
```

Save both. Label them clearly:

```bash
LUKS_UUID=$(sudo blkid -s UUID -o value $PART2)
FS_UUID=$(sudo blkid -s UUID -o value /dev/mapper/root)
BOOT_UUID=$(sudo blkid -s UUID -o value $PART1)

echo "LUKS_UUID=$LUKS_UUID"   # → crypttab.initramfs, kernel cmdline
echo "FS_UUID=$FS_UUID"       # → fstab (/ mount)
echo "BOOT_UUID=$BOOT_UUID"   # → fstab (/boot mount)
```

> **WARNING**: `crypttab.initramfs` and `rd.luks.name=` MUST use the **LUKS container UUID** (`TYPE="crypto_LUKS"`). If you accidentally use the filesystem UUID, systemd will look for a LUKS device that doesn't exist, time out after 90 seconds, and drop to emergency mode.

### Mount subvolumes for chroot

Mount the `@` subvolume as root, then mount remaining subvolumes and EFI:

```bash
# Mount root subvolume with compression
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt

# IMPORTANT: Remount with suid to allow sudo/makepkg inside chroot
sudo mount -o remount,suid /mnt

# Create mountpoints and mount remaining subvolumes
sudo mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache}
sudo mount -o compress=zstd,subvol=@home       /dev/mapper/root /mnt/home
sudo mount -o compress=zstd,subvol=@snapshots   /dev/mapper/root /mnt/.snapshots
sudo mount -o compress=zstd,subvol=@var_log     /dev/mapper/root /mnt/var/log
sudo mount -o compress=zstd,subvol=@var_cache   /dev/mapper/root /mnt/var/cache

# EFI System Partition
sudo mount $PART1 /mnt/boot
```

> **Kali/Debian note:** If `mount --mkdir` is used elsewhere and fails (older util-linux), use `mkdir -p` then `mount` as shown above.

---

## 5. Bootstrap Arch via Chroot

### Pacstrap

**If your host is Arch Linux:**

```bash
sudo pacstrap -K /mnt \
  base \
  linux \
  linux-firmware \
  intel-ucode \
  mkinitcpio \
  cryptsetup \
  dosfstools \
  btrfs-progs \
  networkmanager \
  openssh \
  sudo \
  vim \
  efibootmgr \
  tpm2-tss
```

Optionally add your personal packages to the same `pacstrap` command. Example with a full dev environment:

```bash
sudo pacstrap -K /mnt \
  base linux linux-firmware intel-ucode \
  mkinitcpio cryptsetup dosfstools btrfs-progs \
  networkmanager openssh sudo vim efibootmgr tpm2-tss \
  avahi base-devel bat btop clang curl docker docker-buildx \
  docker-compose eza fastfetch fzf git github-cli gnupg gum \
  inetutils jq jujutsu kitty-terminfo lazydocker lazygit lazyjj \
  less libyaml llvm lua luarocks man-db markdownlint-cli2 mise \
  neovim opencode prettier python-poetry ripgrep rust sqlfluff \
  starship stow stylua tldr tmux tree-sitter whois yazi zellij \
  zoxide zsh
```

> **Kali/Debian note:** If `pacstrap` can't find `pacman`, verify that `pacman-package-manager` was installed via apt (section 2a). The `pacman` binary from the apt package works directly — no symlinks or static binaries needed.

**Package notes:**

- `base` — core system (includes systemd)
- `linux` + `linux-firmware` — kernel + firmware blobs
- `intel-ucode` — **critical** for Intel N150 microcode
- `mkinitcpio` + `cryptsetup` — initramfs + LUKS support
- `btrfs-progs` — btrfs filesystem tools (snapshots, scrub, send/receive)
- `networkmanager` — manages networking after boot (post-LUKS)
- `openssh` — SSH server for the running OS
- `efibootmgr` — UEFI boot entry management
- `tpm2-tss` — TPM2 libraries (for future TPM enrollment)
- `-K` flag — fresh pacman keyring (don't share host keys)

### Generate fstab

**If your host is Arch (or you installed arch-install-scripts per section 2a):**

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

**If `genfstab` is not available** (fallback — write it manually):

```bash
# Use the FILESYSTEM UUID for fstab (NOT the LUKS UUID)
# These were saved in section 4
cat <<EOF | sudo tee /mnt/etc/fstab
# /dev/mapper/root btrfs subvolumes — use FS_UUID (btrfs inside LUKS), NOT LUKS_UUID
UUID=${FS_UUID}    /             btrfs  rw,relatime,compress=zstd,subvol=@            0 0
UUID=${FS_UUID}    /home         btrfs  rw,relatime,compress=zstd,subvol=@home        0 0
UUID=${FS_UUID}    /.snapshots   btrfs  rw,relatime,compress=zstd,subvol=@snapshots   0 0
UUID=${FS_UUID}    /var/log      btrfs  rw,relatime,compress=zstd,subvol=@var_log     0 0
UUID=${FS_UUID}    /var/cache    btrfs  rw,relatime,compress=zstd,subvol=@var_cache   0 0

# EFI System Partition (device name changes between workstation and ZimaBoard — UUID is stable)
UUID=${BOOT_UUID}  /boot  vfat  rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro  0 2
EOF
```

> **Do NOT use the LUKS UUID here.** `fstab` mounts filesystems. All subvolume entries use the same btrfs UUID — they're all on the same filesystem. The LUKS UUID goes in `crypttab.initramfs` (section 6).
>
> **btrfs note:** `fsck` pass (last column) is `0` for btrfs — btrfs does not use traditional `fsck`. Use `btrfs scrub` instead.

**Verify** — should reference UUIDs and subvolumes, not device paths:

```bash
cat /mnt/etc/fstab
```

Expected contents (UUIDs will differ):

```
UUID=xxxxxxxx-xxxx  /             btrfs  rw,relatime,compress=zstd,subvol=@          0 0
UUID=xxxxxxxx-xxxx  /home         btrfs  rw,relatime,compress=zstd,subvol=@home      0 0
UUID=xxxxxxxx-xxxx  /.snapshots   btrfs  rw,relatime,compress=zstd,subvol=@snapshots 0 0
UUID=xxxxxxxx-xxxx  /var/log      btrfs  rw,relatime,compress=zstd,subvol=@var_log   0 0
UUID=xxxxxxxx-xxxx  /var/cache    btrfs  rw,relatime,compress=zstd,subvol=@var_cache  0 0
UUID=XXXX-XXXX      /boot         vfat   rw,relatime,fmask=0022,dmask=0022,...       0 2
```

### Enter chroot

**If `arch-chroot` is available (Arch host, or installed per section 2a):**

```bash
sudo arch-chroot /mnt
```

`arch-chroot` automatically binds `/dev`, `/proc`, `/sys`, `/run` and copies `/etc/resolv.conf`. No manual bind mounts needed (unlike the RPi guide). No qemu-user-static needed (x86_64 → x86_64 is native).

**If `arch-chroot` is not available** (manual chroot from Kali/Debian):

```bash
sudo mount --bind /dev  /mnt/dev
sudo mount --bind /dev/pts /mnt/dev/pts
sudo mount -t proc proc /mnt/proc
sudo mount -t sysfs sys  /mnt/sys
sudo mount --bind /run  /mnt/run
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf
sudo chroot /mnt /bin/bash
```

> **Kali/Debian note on cleanup:** If you used manual bind mounts instead of `arch-chroot`, the cleanup in section 9 (`umount -R /mnt`) will handle everything. The bind mounts are inside `/mnt` and get recursively unmounted.

---

## 6. Configure the System Inside Chroot

### Basic system configuration

```bash
# Timezone (adjust to yours)
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "zimaboard" > /etc/hostname

# Root password
passwd

# Create your user
useradd -m -G wheel -s /bin/bash g
passwd g

# Enable sudo for wheel group
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
chmod 440 /etc/sudoers.d/wheel
```

### Configure /etc/crypttab.initramfs

This tells `sd-encrypt` which LUKS device to unlock at boot. **Use the LUKS container UUID** (`LUKS_UUID` from section 4), NOT the filesystem UUID.

> **Note**: `blkid` inside the chroot may not show the LUKS partition (the raw block device isn't always visible). Use the `LUKS_UUID` you saved in section 4. If you forgot it, run `blkid` on the **host** (outside the chroot) to find the `TYPE="crypto_LUKS"` UUID.

Write the file:

```bash
echo "root  UUID=${LUKS_UUID}  none  luks,discard" > /etc/crypttab.initramfs
```

Result should look like:

```
root  UUID=824f87fe-xxxx-xxxx-xxxx-xxxxxxxxxxxx  none  luks,discard
```

> **CRITICAL**: This is the LUKS container UUID from `blkid $PART2` (shows `TYPE="crypto_LUKS"`). If you use the filesystem UUID from `blkid /dev/mapper/root` (shows `TYPE="btrfs"`), systemd will try to decrypt a non-existent device → timeout → emergency mode → unrecoverable without chroot repair.

The `discard` option enables TRIM through LUKS for NVMe performance (minor security tradeoff — leaks block usage patterns).

### Configure mkinitcpio

`/etc/mkinitcpio.conf`:

```bash
MODULES=(igc)

BINARIES=()

FILES=(/etc/crypttab.initramfs /etc/systemd/network/20-wired.network)

HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-network sd-tinyssh sd-encrypt filesystems fsck)

# Required for sd-tinyssh to trigger the password prompt
SD_TINYSSH_COMMAND="systemd-tty-ask-password-agent --query --watch"
```

**Why `igc` in MODULES**: The Intel i226V NIC driver. The `autodetect` hook detects hardware of the **currently running system** (your workstation), not the ZimaBoard. Explicitly listing `igc` ensures the NIC driver is always included. The fallback UKI (built with `-S autodetect`) includes everything anyway, but the default UKI needs this.

**Hook order explained:**

| Hook          | Purpose                                                |
| ------------- | ------------------------------------------------------ |
| `base`        | Always first                                           |
| `systemd`     | Replaces busybox `udev`. Required for all `sd-*` hooks |
| `autodetect`  | Shrinks initramfs to detected hardware modules         |
| `microcode`   | Intel N150 CPU microcode (embedded in UKI)             |
| `modconf`     | Module config from `/etc/modprobe.d/`                  |
| `kms`         | Kernel mode setting                                    |
| `keyboard`    | Keyboard drivers (for emergency console access)        |
| `sd-vconsole` | Console font/keymap                                    |
| `block`       | Block device drivers (NVMe, SATA, USB storage)         |
| `sd-encrypt`  | LUKS unlock via systemd-cryptsetup                     |
| `filesystems` | ext4, btrfs, etc.                                      |
| `fsck`        | Filesystem check                                       |

> **Note**: `tinyssh` is added later in section 7 after installing the package.

### Configure kernel command line

`/etc/kernel/cmdline` — again, uses the **LUKS container UUID** (same as `crypttab.initramfs`):

```bash
echo "rd.luks.name=${LUKS_UUID}=root root=/dev/mapper/root rootflags=x-systemd.device-timeout=0 rw quiet ip=192.168.100.60::192.168.100.1:255.255.254.0:zimaboard:eth0:none" > /etc/kernel/cmdline
```

Result should look like:

```
rd.luks.name=824f87fe-xxxx-xxxx-xxxx-xxxxxxxxxxxx=root root=/dev/mapper/root rootflags=x-systemd.device-timeout=0 rw quiet ip=192.168.100.60::192.168.100.1:255.255.254.0:zimaboard:eth0:none
```

**Parameters:**

- `rd.luks.name=<LUKS_UUID>=root` — map LUKS device to `/dev/mapper/root` (must match name in `crypttab.initramfs`)
- `root=/dev/mapper/root` — root filesystem is the unlocked LUKS device
- `rootflags=x-systemd.device-timeout=0` — **critical for headless** — wait indefinitely for LUKS unlock. Without this, systemd gives up after 90 seconds and drops to emergency mode (inaccessible remotely).
- `rw` — mount root read-write
- `quiet` — suppress boot messages (remove for debugging first boot)
- `ip=<client-ip>::<gateway-ip>:<netmask>:<hostname>:<device>:none` — **Static IP for initramfs** (required for `sd-network` / `sd-tinyssh`).

> **Verify before moving on**: `cat /etc/kernel/cmdline` and `cat /etc/crypttab.initramfs` should both contain the same UUID.

### Configure UKI preset

Edit `/etc/mkinitcpio.d/linux.preset`:

```bash
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

#default_image="/boot/initramfs-linux.img"
default_uki="/boot/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Key changes from stock:

- Comment out `*_image` lines (no separate initramfs files)
- Set `*_uki` lines pointing to `/boot/EFI/Linux/`
- The fallback UKI skips `autodetect` (includes ALL modules — your safety net for first boot)

### Enable services

```bash
# Networking (post-LUKS)
systemctl enable NetworkManager

# SSH server (post-LUKS)
systemctl enable sshd
```

### Set up SSH for your user

```bash
mkdir -p /home/[user]/.ssh
chmod 700 /home/[user]/.ssh
echo "ssh-ed25519 AAAA... you@workstation" > /home/[user]/.ssh/authorized_keys
chmod 600 /home/[user]/.ssh/authorized_keys
chown -R g:g /home/[user]/.ssh
```

### Harden SSH

`/etc/ssh/sshd_config.d/10-hardening.conf`:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
AuthenticationMethods publickey
```

---

## 7. Headless SSH LUKS Unlock (sd-tinyssh)

### Install Dependencies & Build AUR Hooks

The `sd-network` and `sd-tinyssh` hooks are provided by the `mkinitcpio-systemd-extras` package in the AUR.

Still inside the chroot:

```bash
# As Root: Install base tools and tinyssh
pacman -S --needed base-devel git tinyssh

# As User 'g': Clone and build the systemd-extras package
cd /tmp
git clone https://aur.archlinux.org/mkinitcpio-systemd-extras.git
cd mkinitcpio-systemd-extras
chown -R g:g .
su - g -c "cd /tmp/mkinitcpio-systemd-extras && makepkg -sc"

# As Root: Install the built package
pacman -U mkinitcpio-systemd-extras-*.pkg.tar.zst
```

### Update mkinitcpio hooks

Verify `/etc/mkinitcpio.conf` has the correct hooks (added in Section 6):

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-network sd-tinyssh sd-encrypt filesystems fsck)
```

**What changed:**

- Added `sd-network` for systemd-based networking in initramfs.
- Added `sd-tinyssh` for SSH access in initramfs.
- `sd-tinyssh` works with `systemd-tty-ask-password-agent`.

### Configure tinyssh authorized keys

```bash
mkdir -p /etc/tinyssh

# Copy your public key for initramfs SSH access
echo "ssh-ed25519 AAAA... you@workstation" > /etc/tinyssh/root_key
chmod 600 /etc/tinyssh/root_key
```

### Configure initramfs networking (Static IP)

Create `/etc/systemd/network/20-wired.network` to match your kernel command line:

```ini
[Match]
Name=en* eth*

[Network]
Address=192.168.100.60/23
Gateway=192.168.100.1
```

> **Interface naming**: The ZimaBoard's i226V NICs usually appear as `enp1s0` / `enp2s0`. `en* eth*` covers most cases.

### Generate tinyssh host keys

```bash
mkdir -p /etc/tinyssh/sshkeydir
tinysshd-makekey /etc/tinyssh/sshkeydir
```

> **If AUR installation is problematic in chroot** (GPG key issues, network problems), skip this step and install on first boot. First boot requires keyboard + monitor for passphrase entry. After installing tinyssh and rebuilding initramfs, subsequent boots are headless.

### Update mkinitcpio hooks

Edit `/etc/mkinitcpio.conf` — add `tinyssh` before `sd-encrypt`:

```bash
MODULES=(igc)

BINARIES=()

FILES=(/etc/crypttab.initramfs)

HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block tinyssh sd-encrypt filesystems fsck)
```

**What changed:**

- Added `tinyssh` to HOOKS (before `sd-encrypt`)
- `tinyssh` handles both SSH and networking in initramfs — no separate `sd-network` hook needed

### Configure tinyssh authorized keys

```bash
mkdir -p /etc/tinyssh

# Copy your public key for initramfs SSH access
echo "ssh-ed25519 AAAA... you@workstation" > /etc/tinyssh/root_key
chmod 600 /etc/tinyssh/root_key
```

The tinyssh hook bakes this key into the initramfs. SSH sessions connect as root and present the `systemd-tty-ask-password-agent` prompt for LUKS unlock.

### Configure initramfs networking

Create `/etc/systemd/network/20-wired.network` for initramfs DHCP:

**Option A — DHCP (recommended for first boot):**

```ini
[Match]
Name=en*

[Network]
DHCP=yes
```

**Option B — Static IP:**

```ini
[Match]
Name=enp1s0

[Network]
Address=192.168.100.60/23
Gateway=192.168.100.1
DNS=192.168.100.1
```

> **Interface naming**: Intel i226V NICs appear as `enp1s0` and `enp2s0` on ZimaBoard. `Name=en*` matches both. For static IP, identify correct NIC after first boot (`ip link`) and update.

### Generate tinyssh host keys

```bash
mkdir -p /etc/tinyssh/sshkeydir
tinysshd-makekey /etc/tinyssh/sshkeydir
```

Use **separate keys** from OpenSSH to avoid `known_hosts` conflicts between initramfs and running OS. The `mkinitcpio` output prints the ed25519 fingerprint — save it for client-side verification.

---

## 8. Build UKI and Install Bootloader

### Create UKI output directory

```bash
mkdir -p /boot/EFI/Linux
```

### Install systemd-boot

```bash
bootctl install --no-variables
```

The `--no-variables` flag prevents writing UEFI NVRAM entries on your workstation. It still copies:

- `systemd-bootx64.efi` → `/boot/EFI/systemd/systemd-bootx64.efi`
- `systemd-bootx64.efi` → `/boot/EFI/BOOT/BOOTX64.EFI` (UEFI fallback path)

The **fallback path** is key — the ZimaBoard's UEFI firmware will find and boot it automatically even without an NVRAM entry.

### Configure loader

`/boot/loader/loader.conf`:

```bash
cat <<EOF > /boot/loader/loader.conf
default arch-linux.efi
timeout 3
console-mode auto
editor no
EOF
```

`editor no` prevents boot parameter editing from the menu (security hardening).

> **Verify this file after `bootctl install`** — `bootctl` may overwrite it with a commented-out stub. If `cat /boot/loader/loader.conf` shows only `#timeout 3` / `#console-mode keep`, you must rewrite it.

### Create fallback boot entry (belt-and-suspenders)

UKIs are auto-discovered by systemd-boot from `/boot/EFI/Linux/`. A manual `loader/entries/` config is not strictly required, but provides a fallback if auto-discovery fails:

```bash
mkdir -p /boot/loader/entries

cat <<EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options rd.luks.name=${LUKS_UUID}=root root=/dev/mapper/root rootflags=x-systemd.device-timeout=0 rw quiet ip=192.168.100.60::192.168.100.1:255.255.254.0:zimaboard:eth0:none
EOF
```

> **This entry uses the traditional initramfs (not UKI).** It boots the kernel + initramfs separately. The UKI (`arch-linux.efi`) is the primary boot path. This entry exists as a safety net — if UKI auto-discovery breaks, systemd-boot falls back to `loader/entries/`.

### Build the UKIs

Verify that the systemd-extras hooks are recognized:

```bash
mkinitcpio -L | grep sd-
# Should see sd-network and sd-tinyssh
```

Generate the images:

```bash
mkinitcpio -P
```

This builds both `arch-linux.efi` (default) and `arch-linux-fallback.efi`. Check output for:

- `tinyssh` hook running (copies host keys, prints ed25519 fingerprint)
- `sd-encrypt` hook running
- No errors about missing binaries

> **If tinyssh is not yet installed** (skipped AUR step): just run `mkinitcpio -P` anyway. The default + fallback UKIs will be built without SSH support. You'll use keyboard + monitor for first boot, install tinyssh from the running system, rebuild, and subsequent boots are headless.

### Pre-flight verification checklist

Before exiting chroot, verify everything lines up:

```bash
# 1. crypttab.initramfs uses LUKS UUID (TYPE="crypto_LUKS")
cat /etc/crypttab.initramfs

# 2. kernel cmdline uses same LUKS UUID
cat /etc/kernel/cmdline

# 3. fstab uses FILESYSTEM UUID (TYPE="btrfs") for / and subvolumes
cat /etc/fstab

# 4. loader.conf has correct default
cat /boot/loader/loader.conf

# 5. UKI files exist on EFI partition
ls -lh /boot/EFI/Linux/

# 6. Fallback boot entry exists
cat /boot/loader/entries/arch.conf
```

All UUID references in items 1, 2, and 6 must be identical (LUKS UUID). Item 3 must use a different UUID (filesystem UUID).

### Verify UKI contents (optional)

```bash
ls -lh /boot/EFI/Linux/
# arch-linux.efi          ~40-80 MB (kernel + initramfs + microcode)
# arch-linux-fallback.efi ~80-150 MB (all modules)
```

---

## 9. Exit Chroot and Clean Up

```bash
# Inside chroot:
sync
exit

# Back on workstation:
sudo umount -R /mnt
sudo cryptsetup close root
```

If the NVMe is in a USB enclosure, you can now safely disconnect it.

---

## 10. First Boot on ZimaBoard 2

### Hardware setup

1. Install the Samsung 990 Pro into the PCIe-to-M.2 adapter card
2. Insert the adapter card into the ZimaBoard 2's PCIe slot
3. Connect Ethernet to one of the 2.5GbE ports
4. Connect Mini DisplayPort adapter + monitor
5. Connect USB keyboard
6. Plug in power adapter (board powers on immediately)

### Change BIOS boot order

1. Press **Delete** repeatedly when the Zima logo appears to enter BIOS
2. Navigate to **Boot** tab
3. Set **Boot Option #1** to the NVMe drive (should appear as "UEFI: Samsung 990 Pro" or similar)
4. _(Optional)_ While in BIOS: go to Security/Trusted Computing and **enable Intel PTT** (for future TPM enrollment)
5. Press **F10** to Save & Exit

### First boot

The system boots into systemd-boot → loads the fallback UKI → initramfs starts.

**If tinyssh was installed:**

1. Wait ~15-30 seconds for initramfs to configure networking
2. From your workstation:

   ```bash
   ssh root@<ZIMABOARD-IP>
   ```

   > tinyssh listens on port 22 by default (not 2222 like dropbear). Adjust if you changed `tinyssh@.socket`.

3. You'll see the passphrase prompt from `systemd-tty-ask-password-agent`
4. Type your LUKS passphrase, press Enter
5. Connection closes automatically
6. Wait a few seconds, then SSH into the running OS:

   ```bash
   ssh g@<ZIMABOARD-IP>
   ```

**If tinyssh was NOT installed:**

1. The LUKS passphrase prompt appears on the monitor
2. Type your passphrase on the USB keyboard
3. System boots into Arch Linux
4. SSH in from your workstation: `ssh g@<ZIMABOARD-IP>`
5. Install tinyssh now (see post-boot section below)

### Create proper UEFI boot entry

After first successful boot, create a persistent NVRAM entry:

```bash
sudo bootctl install
```

This time it runs on the actual ZimaBoard hardware and creates a proper UEFI boot entry.

---

## 11. Post-Boot Configuration

### Regenerate initramfs (important!)

The default UKI was built on your workstation — `autodetect` detected your workstation's hardware. Rebuild on the ZimaBoard so it detects the correct hardware:

```bash
sudo mkinitcpio -P
```

### Install tinyssh (if skipped during chroot)

```bash
# Install AUR helper
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay-bin.git /tmp/yay-bin
cd /tmp/yay-bin && makepkg -si

# Install mkinitcpio-tinyssh
yay -S mkinitcpio-tinyssh

# Apply configuration from section 7 (mkinitcpio.conf, network config,
# authorized_keys, host keys), then rebuild:
sudo mkinitcpio -P
```

### btrfs snapshot workflow for safe updates

The main reason to use btrfs: snapshot before updates, rollback if something breaks — all remotely via SSH.

**Before any system update:**

```bash
# Create a read-only snapshot of root
sudo btrfs subvolume snapshot -r / /.snapshots/@_pre-update_$(date +%Y%m%d-%H%M)
```

**Update the system:**

```bash
sudo pacman -Syu
sudo mkinitcpio -P   # if kernel updated
```

**If update breaks something — rollback:**

```bash
# From a working SSH session (or chroot from workstation):
# 1. Mount the raw btrfs volume
sudo mount -o subvolid=5 /dev/mapper/root /mnt/btrfs-root

# 2. Move broken @ out of the way
sudo mv /mnt/btrfs-root/@ /mnt/btrfs-root/@_broken

# 3. Snapshot the good pre-update snapshot as the new writable @
sudo btrfs subvolume snapshot /mnt/btrfs-root/@snapshots/@_pre-update_YYYYMMDD-HHMM /mnt/btrfs-root/@

# 4. Reboot
sudo reboot
```

**Cleanup old snapshots:**

```bash
# List snapshots
sudo btrfs subvolume list -s /

# Delete old ones
sudo btrfs subvolume delete /.snapshots/@_pre-update_20260401-1200
```

> **Tip**: For automated snapshots, consider installing `snapper` or `timeshift` after first boot. Both integrate with pacman hooks to auto-snapshot before every update.

### Set hostname for mDNS discovery

```bash
sudo hostnamectl set-hostname zimaboard
```

If using NetworkManager, mDNS should work out of the box (`zimaboard.local`).

### Optional: Enroll TPM for auto-unlock

With Intel PTT enabled in BIOS, you can add TPM auto-unlock so normal reboots don't require SSH passphrase entry:

```bash
# Create recovery key FIRST
sudo systemd-cryptenroll /dev/nvme0n1p2 --recovery-key
# SAVE THE PRINTED KEY

# Enroll TPM
sudo systemd-cryptenroll /dev/nvme0n1p2 --tpm2-device=auto --tpm2-pcrs=7
```

Update `/etc/crypttab.initramfs`:

```
root  UUID=<YOUR-LUKS-UUID>  none  luks,discard,tpm2-device=auto,token-timeout=30
```

Rebuild initramfs:

```bash
sudo mkinitcpio -P
```

Now the boot flow is: TPM auto-unlocks (no interaction) → if TPM fails → falls back to passphrase prompt after 30s → SSH in to enter passphrase via tinyssh.

### Optional: Install your environment (omaterm, etc.)

```bash
# omaterm is a post-install provisioner — run it after the base system is solid
bash <(curl -fsSL https://raw.githubusercontent.com/omacom-io/omaterm/main/install.sh)
```

---

## 12. Gotchas and Lessons Learned

### LUKS UUID vs filesystem UUID (the #1 boot killer)

There are two UUIDs for an encrypted root. Confusing them causes `Timed out waiting for device /dev/disk/by-uuid/...` → cascade of `Dependency failed` → emergency mode.

| UUID           | Source                                                | Used in                                                              |
| -------------- | ----------------------------------------------------- | -------------------------------------------------------------------- |
| LUKS container | `blkid /dev/sdX2` (`TYPE="crypto_LUKS"`)              | `crypttab.initramfs`, `/etc/kernel/cmdline`, `loader/entries/*.conf` |
| Filesystem     | `blkid /dev/mapper/root` (`TYPE="ext4"` or `"btrfs"`) | `fstab` (mount entries)                                              |

**Three files must all use the LUKS UUID:**

1. `/etc/crypttab.initramfs` — tells `sd-encrypt` which device to unlock
2. `/etc/kernel/cmdline` — gets **baked into the UKI** by `mkinitcpio`. If wrong here, the UKI boots with the wrong UUID even if `crypttab.initramfs` is correct
3. `/boot/loader/entries/arch.conf` — fallback boot entry

**Symptom**: Boot shows `Timed out waiting for device /dev/disk/by-uuid/<wrong-uuid>`, then `Dependency failed for Cryptography Setup for root`, then emergency mode with locked root account.

> **Lesson learned**: Fixing `crypttab.initramfs` alone is not enough. The UKI embeds `/etc/kernel/cmdline` at build time. If `kernel/cmdline` has the wrong UUID, the UKI carries that error even after rebuilding. Always verify with `strings /boot/EFI/Linux/arch-linux.efi | grep rd.luks.name` after `mkinitcpio -P`.

**Fix from workstation**: Mount LUKS, fix both `crypttab.initramfs` AND `/etc/kernel/cmdline` to use LUKS UUID, rebuild initramfs via chroot. See section 14 for chroot procedure.

### Device names change between workstation and ZimaBoard

The NVMe is `/dev/sdb` in a USB enclosure, `/dev/nvme1n1` in a workstation M.2 slot, and `/dev/nvme0n1` in the ZimaBoard.

**Every reference must use UUIDs**: `genfstab -U`, `rd.luks.name=UUID=`, `blkid` for `/etc/crypttab.initramfs`. If you accidentally use device paths anywhere, the system won't boot on the ZimaBoard.

### Empty `loader/entries/` directory

`bootctl install` does NOT create boot entries. It only copies the bootloader EFI binaries. You must either:

1. Build UKIs (auto-discovered from `/boot/EFI/Linux/`) — primary method in this guide
2. Create `loader/entries/arch.conf` manually — fallback method

If both are missing, systemd-boot shows an empty menu.

### `loader.conf` overwritten by `bootctl`

`bootctl install` may overwrite `loader.conf` with a commented-out stub (`#timeout 3`, `#console-mode keep`). Always verify `cat /boot/loader/loader.conf` after running `bootctl install` and rewrite if needed.

### `autodetect` hook and chroot builds

The `autodetect` hook detects hardware of the system building the initramfs (your workstation). This means the default UKI may be missing ZimaBoard-specific modules (e.g., `igc` for the i226V NIC).

**Mitigations:**

- Explicitly list `igc` in `MODULES=(igc)` in mkinitcpio.conf
- The **fallback UKI** includes ALL modules (skips autodetect) — this is your safety net
- After first boot on the ZimaBoard, rebuild initramfs so autodetect picks up the correct hardware

### UEFI boot entries and disk migration

UEFI NVRAM entries are stored in the motherboard firmware, not on disk. When you move the NVMe from workstation to ZimaBoard, the boot entries don't follow. The ZimaBoard boots from the **UEFI fallback path** (`EFI/BOOT/BOOTX64.EFI`) which `bootctl install` placed there. After first boot, run `bootctl install` again to create a proper NVRAM entry.

### The 90-second death trap

Without `rootflags=x-systemd.device-timeout=0`, systemd waits only 90 seconds for the root device. If LUKS isn't unlocked by then, it enters emergency mode — which you can't access remotely. **Always set the infinite timeout for headless LUKS.**

### systemd-boot `editor no`

Set `editor no` in `loader.conf` to prevent boot parameter editing from the physical console. Without this, anyone with keyboard access can add `init=/bin/bash` at the boot menu.

### Two NICs — which one to use?

The ZimaBoard 2 has two i226V NICs. The network config with `Name=en*` matches both. DHCP will work on whichever is connected. For static IP, identify the correct NIC name after first boot (`ip link`) and update the config.

### Mini DisplayPort adapter required

The ZimaBoard 2 only has Mini DisplayPort out. You **must** have a MiniDP-to-HDMI (or MiniDP-to-DP) adapter for initial BIOS setup. IceWhale does not include one in the box.

### eMMC coexistence

The 64GB eMMC ships with ZimaOS. You can leave it — just change the boot order in BIOS to prioritize the NVMe. The two OSes don't interfere (different bootloaders, different ESPs). You can switch back to ZimaOS by pressing F11 at boot and selecting the eMMC.

### PCIe x2 lane limitation

Despite the "PCIe 3.0 x4" marketing, the ZimaBoard 2's PCIe slot provides **x2 electrical lanes** (~2 GB/s). Your 990 Pro will run at about 1/4 its rated speed. Still much faster than eMMC or SATA.

### btrfs-specific gotchas

**No traditional `fsck`.** btrfs does not use `fsck` at boot. Set the last field in `fstab` to `0` for all btrfs entries. Use `btrfs scrub start /` periodically for data integrity checks.

**Swap file on btrfs.** Swap files require special handling on btrfs. Create with `nocow` attribute:

```bash
sudo btrfs filesystem mkswapfile --size 4g /swap/swapfile
sudo swapon /swap/swapfile
```

Or use a swap partition instead (simpler).

**Snapshot storage grows.** Snapshots are cheap initially but grow as the filesystem diverges from the snapshot. Clean up old snapshots regularly. A snapshot from 6 months ago with 50GB of package updates in between is not "free".

**`genfstab` and subvolumes.** `genfstab -U` generates correct subvolume entries if all subvolumes are mounted before running it. If you write fstab manually, ensure every subvolume has its `subvol=` option — omitting it mounts the top-level volume (subvolid=5) which is NOT your `@` subvolume.

### Non-Arch host (Kali/Debian) specific gotchas

If you're building from Kali, Debian, or another non-Arch host, watch out for these:

**Keyring initialization failures.** The #1 problem. `pacstrap` verifies package signatures using the Arch Linux keyring. If you get errors like `invalid or corrupted package (PGP signature)` or `key could not be looked up remotely`, reinitialize:

```bash
sudo rm -rf /etc/pacman.d/gnupg
sudo pacman-key --init
sudo pacman-key --populate archlinux
```

**`mount --mkdir` not available.** Older Debian/Kali kernels may not support `mount --mkdir`. If it fails:

```bash
sudo mkdir -p /mnt/boot
sudo mount $PART1 /mnt/boot
```

**DNS resolution inside chroot.** `arch-chroot` copies `/etc/resolv.conf` from the host. On Kali with `systemd-resolved`, this file is often a symlink to a stub resolver (`127.0.0.53`). The chroot can't reach that stub. Fix before entering chroot:

```bash
sudo rm /mnt/etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /mnt/etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /mnt/etc/resolv.conf
```

**`hwclock --systohc` may warn or fail in chroot.** Harmless — trying to write to hardware RTC from chroot. Clock gets set properly on ZimaBoard at first boot.

**Everything inside the chroot is Arch.** Once you're past `pacstrap` and inside the chroot, you're in a full Arch environment. `pacman`, `mkinitcpio`, `bootctl`, `systemctl enable` — all run using Arch binaries in the chroot, not host Debian tools. Host distro is irrelevant from section 6 onward.

---

## 13. SSH Config for Key Separation

The initramfs tinyssh and the running OS OpenSSH have different host keys. Configure separate entries on your client:

`~/.ssh/config`:

```
Host zimaboard
    Hostname zimaboard.local
    User g
    IdentityFile ~/.ssh/id_ed25519

Host zimaboard-unlock
    Hostname zimaboard.local
    User root
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    HostKeyAlias zimaboard-unlock
```

> **Port note**: tinyssh defaults to port 22 (unlike dropbear which uses 2222). The `HostKeyAlias` keeps the initramfs and running OS host keys separate in `known_hosts`. If you changed the tinyssh socket unit to a different port, update accordingly.

For static IP in initramfs:

```
Host zimaboard-unlock
    Hostname 192.168.100.60
    User root
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    HostKeyAlias zimaboard-unlock
```

---

## 14. Quick Reference

### Unlock procedure (normal operation)

```bash
# If TPM is enrolled — system boots automatically, no action needed

# If TPM fails or not enrolled — SSH unlock via tinyssh:
ssh zimaboard-unlock
# Type LUKS passphrase at prompt
# Connection closes, system boots

# Then connect to running OS:
ssh zimaboard
```

### Re-mount for chroot (from workstation)

**Arch host:**

```bash
# Connect NVMe to workstation
PART2=/dev/sdb2          # adjust for your setup
sudo cryptsetup open $PART2 root
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
sudo mount /dev/sdb1 /mnt/boot    # adjust partition device
sudo arch-chroot /mnt

# When done:
exit
sudo umount -R /mnt
sudo cryptsetup close root
```

**Kali/Debian host (manual chroot):**

```bash
# Connect NVMe to workstation
PART2=/dev/sdb2          # adjust for your setup
sudo cryptsetup open $PART2 root
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
sudo mount /dev/sdb1 /mnt/boot    # adjust partition device

# Bind mounts
sudo mount --bind /dev     /mnt/dev
sudo mount --bind /dev/pts /mnt/dev/pts
sudo mount -t proc proc    /mnt/proc
sudo mount -t sysfs sys    /mnt/sys
sudo mount --bind /run     /mnt/run

# Fix DNS if needed
sudo rm /mnt/etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /mnt/etc/resolv.conf

sudo chroot /mnt /bin/bash

# When done:
exit
sudo umount -R /mnt
sudo cryptsetup close root
```

### Rebuild initramfs (on the ZimaBoard)

```bash
sudo mkinitcpio -P
```

### Emergency chroot repair (UUID mismatch or boot failure)

If the system drops to emergency mode with `Timed out waiting for device`, the most likely cause is wrong UUID in `crypttab.initramfs`. Fix from workstation:

```bash
# 1. Connect drive to workstation, identify partitions
lsblk -f
# Find the LUKS partition (TYPE="crypto_LUKS") and note its UUID

# 2. Open LUKS and mount root subvolume
PART2=/dev/sdX2          # adjust
sudo cryptsetup open $PART2 root
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt

# 3. Mount EFI partition at /boot (NOT /root or elsewhere)
PART1=/dev/sdX1          # adjust
sudo mount $PART1 /mnt/boot

# 4. Verify UUIDs match
LUKS_UUID=$(sudo blkid -s UUID -o value $PART2)
echo "LUKS UUID: $LUKS_UUID"
cat /mnt/etc/crypttab.initramfs    # must contain $LUKS_UUID
cat /mnt/etc/kernel/cmdline        # must contain $LUKS_UUID

# 5. Fix if wrong
echo "root  UUID=${LUKS_UUID}  none  luks,discard" > /mnt/etc/crypttab.initramfs

# 6. Chroot and rebuild
sudo arch-chroot /mnt
mkinitcpio -P
cat /boot/loader/loader.conf       # verify not overwritten
exit

# 7. Cleanup
sudo umount -R /mnt
sudo cryptsetup close root
```

---

## 15. Resources

### Arch Wiki

- [Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Install from existing Linux](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux)
- [dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
- [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
- [Unified kernel image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)

### Packages

- [mkinitcpio-tinyssh (AUR)](https://aur.archlinux.org/packages/mkinitcpio-tinyssh)
- [systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll)

### Cross-Distro Bootstrap (for non-Arch hosts)

- [Arch Wiki — Install from existing Linux](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux) — official docs for cross-distro installs
- Kali/Debian: `sudo apt install pacman-package-manager archlinux-keyring arch-install-scripts`

### ZimaBoard 2

- [ZimaBoard 2 Arch install (official)](https://www.zimaspace.com/docs/zimaboard/Arch-Linux-Installation-on-ZimaBoard-2)
- [PCIe ecosystem FAQ](https://shop.zimaspace.com/pages/pcie-x4-interface)
- [ZimaBoard 2 getting started](https://www.zimaspace.com/docs/zimaboard/Power-on-Zimaboard2)

### Companion documents

- [ZimaBoard 2 physical security hardening](zimaboard2-physical-security-hardening.md) — Secure Boot, TPM, USB/console lockdown
- [RPi 5 headless LUKS setup](final-headless-luks-instructions-public.md) — similar guide for Raspberry Pi

---

## Appendix A: Shrink, Image, Restore, Expand

Create a compact disk image from a large drive so you can store, transfer, and restore the installation without imaging terabytes of empty space.

**The problem:** Your 1.8TB drive has ~10GB of actual data. A raw `dd` image would be 1.8TB. By shrinking the filesystem → LUKS → partition first, you image only the used portion.

**The chain (innermost to outermost):**

```
partition (sde2) → LUKS container → btrfs filesystem
```

Shrink order: btrfs → LUKS → partition (inside out)
Expand order: partition → LUKS → btrfs (outside in)

### Prerequisites

```bash
sudo apt install gdisk cryptsetup btrfs-progs
```

> **All operations below run on the workstation (Kali/Debian), NOT inside chroot.** The filesystems must be **unmounted** for resize operations.

---

### A1. Shrink (before imaging)

#### Step 1: Open LUKS (do NOT mount)

```bash
DISK=/dev/sde              # adjust for your setup
PART1=${DISK}1             # EFI
PART2=${DISK}2             # LUKS
sudo cryptsetup open $PART2 root
```

#### Step 2: Shrink btrfs filesystem

btrfs can resize while mounted. Mount temporarily, shrink, unmount:

```bash
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt

# Check current usage
sudo btrfs filesystem usage /mnt

# Shrink to used space + 1GB headroom
# Example: if used is 8GB, shrink to 10GB
TARGET_FS_SIZE=10G
sudo btrfs filesystem resize $TARGET_FS_SIZE /mnt

sudo umount /mnt
```

> **btrfs resize uses the mounted path**, not the device. The size argument is the **target size**, not the amount to shrink by. Use `btrfs filesystem usage` to determine actual usage first.

#### Step 3: Shrink LUKS container

`cryptsetup resize` shrinks the LUKS mapping to match the filesystem. The size must be **at least** as large as the btrfs filesystem:

```bash
# Convert target to sectors (512-byte sectors)
# 10GB = 10 * 1024 * 1024 * 2 = 20971520 sectors
# Or use MB: 10240MB * 2048 = 20971520 sectors
TARGET_SECTORS=$((10 * 1024 * 1024 * 2))

sudo cryptsetup resize root --size $TARGET_SECTORS
```

> **CRITICAL:** The LUKS size must be >= btrfs size. If you make it smaller, you corrupt the filesystem. When in doubt, add extra margin.

#### Step 4: Close LUKS

```bash
sudo cryptsetup close root
```

#### Step 5: Shrink the partition

Use `sgdisk` or `gdisk` to shrink partition 2. `sgdisk` cannot resize in-place — delete and recreate with the same start sector but smaller end:

```bash
# Record the start sector of partition 2
START=$(sudo sgdisk -i 2 $DISK | grep "First sector" | awk '{print $3}')
echo "Partition 2 starts at sector: $START"

# Calculate new end sector
# LUKS header is 16MB (32768 sectors). Add to target:
# Total partition size = LUKS overhead + filesystem
# Use slightly more than TARGET_SECTORS to account for LUKS header
LUKS_OVERHEAD=32768
PART_SECTORS=$((TARGET_SECTORS + LUKS_OVERHEAD))
END=$((START + PART_SECTORS - 1))

echo "New end sector: $END"

# Delete partition 2 and recreate smaller
# WARNING: This preserves data ONLY if start sector stays the same
sudo sgdisk --delete=2 $DISK
sudo sgdisk --new=2:${START}:${END} --typecode=2:8304 --change-name=2:"cryptroot" $DISK

# Verify
sudo sgdisk -p $DISK
```

> **WARNING:** Triple-check `$START` matches the original. If the start sector changes, LUKS header is lost and data is destroyed.

---

### A2. Create the image

Now image only the used portion of the disk (EFI + shrunk root partition):

```bash
# Calculate how many bytes to image
# Image from start of disk through end of partition 2
END_SECTOR=$(sudo sgdisk -i 2 $DISK | grep "Last sector" | awk '{print $3}')
# Add 34 sectors for backup GPT at end of imaged area
IMAGE_SECTORS=$((END_SECTOR + 34))
IMAGE_BYTES=$((IMAGE_SECTORS * 512))

echo "Image size: $((IMAGE_BYTES / 1024 / 1024)) MB"

# Create compressed image
sudo dd if=$DISK bs=1M count=$((IMAGE_BYTES / 1024 / 1024 + 1)) status=progress | \
  zstd -T0 -10 > ~/zimaboard-arch-$(date +%Y%m%d).img.zst
```

> **Why `dd` and not `partclone`?** `dd` captures the partition table, EFI partition, and LUKS container as one unit. Restoring is a single operation. The tradeoff is imaging free space within the shrunken partition — acceptable at 10-15GB.

#### Verify the image

```bash
# Check image size
ls -lh ~/zimaboard-arch-*.img.zst

# Optional: verify by mounting via loopback
zstd -d ~/zimaboard-arch-*.img.zst --stdout | sudo dd of=/dev/null status=progress
```

---

### A3. Restore image to a new drive

```bash
NEW_DISK=/dev/sdX           # adjust — THE NEW TARGET DRIVE

# WARNING: This destroys everything on $NEW_DISK
zstd -d ~/zimaboard-arch-*.img.zst --stdout | \
  sudo dd of=$NEW_DISK bs=1M status=progress

# Fix backup GPT (sgdisk warns about this after partial-disk restore)
sudo sgdisk -e $NEW_DISK
```

---

### A4. Expand back to full drive size

After restoring to a new (possibly larger) drive, expand everything back.

#### Step 1: Expand partition 2 to fill remaining space

```bash
# Record start sector (must not change)
START=$(sudo sgdisk -i 2 $NEW_DISK | grep "First sector" | awk '{print $3}')

# Delete and recreate partition 2 using all remaining space
sudo sgdisk --delete=2 $NEW_DISK
sudo sgdisk --new=2:${START}:0 --typecode=2:8304 --change-name=2:"cryptroot" $NEW_DISK

# Verify
sudo sgdisk -p $NEW_DISK
```

#### Step 2: Open and expand LUKS

```bash
PART2=${NEW_DISK}2          # or ${NEW_DISK}p2 for NVMe
sudo cryptsetup open $PART2 root

# Resize LUKS to fill the partition (no --size = use all available)
sudo cryptsetup resize root
```

#### Step 3: Expand btrfs to fill LUKS

```bash
# Mount the root subvolume
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt

# Expand btrfs to fill container
sudo btrfs filesystem resize max /mnt

# Verify
sudo btrfs filesystem usage /mnt

sudo umount /mnt
sudo cryptsetup close root
```

---

### A5. Quick reference (copy-paste)

**Shrink + image (from workstation):**

```bash
DISK=/dev/sde
TARGET_FS=10G

# Shrink
sudo cryptsetup open ${DISK}2 root
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
sudo btrfs filesystem resize $TARGET_FS /mnt
sudo umount /mnt
TARGET_SECTORS=$((10 * 1024 * 1024 * 2))
sudo cryptsetup resize root --size $TARGET_SECTORS
sudo cryptsetup close root
START=$(sudo sgdisk -i 2 $DISK | grep "First sector" | awk '{print $3}')
END=$((START + TARGET_SECTORS + 32768 - 1))
sudo sgdisk --delete=2 $DISK
sudo sgdisk --new=2:${START}:${END} --typecode=2:8304 --change-name=2:"cryptroot" $DISK

# Image
END_SECTOR=$(sudo sgdisk -i 2 $DISK | grep "Last sector" | awk '{print $3}')
IMAGE_MB=$(( (END_SECTOR + 34) * 512 / 1024 / 1024 + 1 ))
sudo dd if=$DISK bs=1M count=$IMAGE_MB status=progress | zstd -T0 -10 > ~/zima-arch-$(date +%Y%m%d).img.zst
```

**Restore + expand (to new drive):**

```bash
NEW_DISK=/dev/sdX

# Restore
zstd -d ~/zima-arch-*.img.zst --stdout | sudo dd of=$NEW_DISK bs=1M status=progress
sudo sgdisk -e $NEW_DISK

# Expand
START=$(sudo sgdisk -i 2 $NEW_DISK | grep "First sector" | awk '{print $3}')
sudo sgdisk --delete=2 $NEW_DISK
sudo sgdisk --new=2:${START}:0 --typecode=2:8304 --change-name=2:"cryptroot" $NEW_DISK
sudo cryptsetup open ${NEW_DISK}2 root
sudo cryptsetup resize root
sudo mount -o compress=zstd,subvol=@ /dev/mapper/root /mnt
sudo btrfs filesystem resize max /mnt
sudo btrfs filesystem usage /mnt
sudo umount /mnt
sudo cryptsetup close root
```

> **After restore + expand:** UUIDs are preserved from the original. `fstab`, `crypttab.initramfs`, and kernel cmdline all remain valid. No chroot rebuild needed unless you changed partitions.
