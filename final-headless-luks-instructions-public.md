# Raspberry Pi 5: Headless LUKS-Encrypted Root with SSH Unlock (2026)

> **Last tested**: April 2026, Raspberry Pi OS Lite 64-bit (Bookworm/Trixie), kernel 6.12.x
> **Platform**: Raspberry Pi 5 Model B
> **Host**: Any x86_64 Linux with `qemu-user-static` (Arch, Debian, Ubuntu, Fedora, etc.)

Most existing guides for encrypted Pi setups are outdated and broken on the Pi 5. This guide has been battle-tested through a week of debugging and documents every gotcha encountered along the way.

---

## What This Does

Takes a stock Raspberry Pi OS Lite 64-bit image and creates a fully encrypted root partition with:
- **LUKS2 encryption** on the root partition
- **Dropbear SSH** in the initramfs for remote unlock
- **Static IP** during initramfs so you can SSH in before the root is decrypted
- **Headless operation** — no monitor/keyboard needed, ever

The boot partition remains unencrypted (required by the Pi firmware).

---

## Prerequisites

- A Raspberry Pi OS Lite 64-bit image (already first-booted or configured via rpi-imager with user/WiFi/SSH)
- A Linux computer (host) for image preparation
- An SSH key pair (ed25519 recommended)

---

## 1. Host Setup

### Install dependencies

**Arch Linux:**
```bash
sudo pacman -S parted multipath-tools cryptsetup rsync qemu-user-static qemu-user-static-binfmt
```

**Debian/Ubuntu:**
```bash
sudo apt install -y parted kpartx cryptsetup-bin rsync binfmt-support qemu-user-static
```

### Prepare images

Create two copies — one pristine base (read-only source), one target (will be encrypted):

```bash
cp pi-base.img pi-encrypted.img
```

Expand the target to accommodate LUKS overhead:

```bash
dd if=/dev/zero bs=1G count=1 >> pi-encrypted.img
parted pi-encrypted.img resizepart 2 100%
```

### Map loop devices

```bash
sudo kpartx -ar pi-base.img        # read-only
sudo kpartx -a  pi-encrypted.img
```

Identify your loop devices:

```bash
lsblk
# Example output:
# loop0 → pi-base.img       (loop0p1 = boot, loop0p2 = root)
# loop1 → pi-encrypted.img  (loop1p1 = boot, loop1p2 = root)
```

> Adjust loop device numbers in all commands below to match your system.

### Encrypt the target root partition

```bash
sudo cryptsetup luksFormat /dev/mapper/loop1p2
# Enter and confirm your passphrase
```

> **Pi 5 note**: The default `aes-xts-plain64` algorithm is fast on Pi 5 thanks to hardware AES acceleration. For older Pis without hardware AES, use `--cipher xchacha20,aes-adiantum-plain64` instead.
>
> **RAM note**: cryptsetup benchmarks your host for PBKDF settings. If your host has more RAM than your Pi, add `--pbkdf-memory 512000 --pbkdf-parallel=1` to prevent creating a header that the Pi can't unlock.

Open, format, and mount:

```bash
sudo cryptsetup open /dev/mapper/loop1p2 crypted
sudo mkfs.ext4 /dev/mapper/crypted
sudo mkdir -p /mnt/chroot
sudo mount /dev/mapper/crypted /mnt/chroot
```

### Copy root filesystem

```bash
sudo mkdir -p /mnt/original
sudo mount /dev/mapper/loop0p2 /mnt/original

sudo rsync --archive --hard-links --acls --xattrs --one-file-system \
  --numeric-ids --info="progress2" /mnt/original/* /mnt/chroot/
```

### Note the LUKS UUID

```bash
sudo blkid /dev/mapper/loop1p2
# UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS"
```

Save this UUID — you'll need it in several config files.

### Mount boot partition and enter chroot

```bash
sudo mkdir -p /mnt/chroot/boot
sudo mount /dev/mapper/loop1p1 /mnt/chroot/boot

sudo cp /etc/resolv.conf /mnt/chroot/etc/resolv.conf

for i in dev proc sys run; do
  sudo mount --rbind /$i /mnt/chroot/$i
  sudo mount --make-rslave /mnt/chroot/$i
done

sudo LANG=C chroot /mnt/chroot /bin/bash
```

---

## 2. Inside the Chroot

### Install packages

```bash
apt update
apt install -y busybox cryptsetup cryptsetup-initramfs dropbear-initramfs
```

### Configure /etc/fstab

Replace the root partition line (leave the boot line unchanged):

```bash
# Change this:
PARTUUID=xxxxxxxx-02  /  ext4  defaults,noatime  0  1
# To this:
/dev/mapper/crypted / ext4 defaults,noatime 0 1
```

### Configure /etc/crypttab

Create or edit `/etc/crypttab`:

```
crypted UUID=<YOUR-LUKS-UUID> none luks,initramfs
```

The `initramfs` option is critical — without it, the entry won't be included in the initrd.

### Configure cryptsetup-initramfs

Edit `/etc/cryptsetup-initramfs/conf-hook`. Uncomment or add:

```bash
CRYPTSETUP=y
```

**Critical for chroot builds**: The cryptroot hook can't detect the LUKS device inside a chroot. Hardcode the values by appending to the same file:

```bash
export CRYPTSETUP=y
_CRYPTTAB_NAME="crypted"
_CRYPTTAB_SOURCE="UUID=<YOUR-LUKS-UUID>"
_CRYPTTAB_OPTIONS="luks,initramfs"
```

### Patch cryptroot hook

The initramfs cryptroot hook resolves UUIDs to `/dev/mapper/loopXpY` paths during build. These won't exist on the Pi. This patch preserves the UUID:

```bash
patch --no-backup-if-mismatch /usr/share/initramfs-tools/hooks/cryptroot << 'EOF'
--- cryptroot
+++ cryptroot
@@ -33,7 +33,7 @@
         printf '%s\0' "$target" >>"$DESTDIR/cryptroot/targets"
         crypttab_find_entry "$target" || return 1
         crypttab_parse_options --missing-path=warn || return 1
-        crypttab_print_entry
+        printf '%s %s %s %s\n' "$_CRYPTTAB_NAME" "$_CRYPTTAB_SOURCE" "$_CRYPTTAB_KEY" "$_CRYPTTAB_OPTIONS" >&3
     fi
 }
EOF
```

### Increase unlock timeout

The default 10-second timeout is too short:

```bash
sed -i 's/^TIMEOUT=.*/TIMEOUT=100/g' /usr/share/cryptsetup/initramfs/bin/cryptroot-unlock
```

### Configure dropbear SSH

Edit `/etc/dropbear/initramfs/dropbear.conf`:

```bash
DROPBEAR=y
```

Add your SSH public key:

```bash
echo "ssh-ed25519 AAAA... you@host" > /etc/dropbear/initramfs/authorized_keys
chmod 0600 /etc/dropbear/initramfs/authorized_keys
```

### Configure initramfs networking

Edit `/etc/initramfs-tools/initramfs.conf`:

```bash
MODULES=most
DEVICE=<YOUR-INTERFACE>
IP=<YOUR-IP>::<YOUR-GATEWAY>:<YOUR-NETMASK>:<YOUR-HOSTNAME>:<YOUR-INTERFACE>:off
```

Example with static IP:
```bash
MODULES=most
DEVICE=eth0
IP=192.168.1.50::192.168.1.1:255.255.255.0:pi-unlock:eth0:off
```

> **CRITICAL — `ip=` format gotcha**: The format is `ip=client:server:gateway:netmask:hostname:device:autoconf`. The **device field** (field 6) must be a real interface name (`eth0`) or **left empty**. Do **NOT** put `any` in the device field — it gets interpreted as a literal device name and the initramfs will wait 180 seconds for `/sys/class/net/any` which never appears, resulting in no network.
>
> - Static IP on eth0: `ip=192.168.1.50::192.168.1.1:255.255.255.0:hostname:eth0:off`
> - DHCP on any interface: `ip=::::hostname::dhcp` (device field empty)
> - DHCP on specific interface: `ip=::::hostname:eth0:dhcp`

Add kernel modules to `/etc/initramfs-tools/modules`:

```
# Encryption
aes_ce_blk
aes_ce_cipher
sha2_ce
sha1_ce
dm_crypt

# Network (may be built-in on 2712 kernel, but harmless to list)
pcie_brcmstb
broadcom
genet
```

### Configure post-boot networking (optional)

If you want LAN prioritized over WiFi after the full OS boots, create `/etc/NetworkManager/system-connections/eth0.nmconnection`:

```ini
[connection]
id=eth0
type=ethernet
interface-name=eth0

[ethernet]

[ipv4]
method=auto
route-metric=100

[ipv6]
addr-gen-mode=default
method=auto
```

`route-metric=100` gives wired Ethernet priority over WiFi (default WiFi metric is ~600).

Set permissions:
```bash
chmod 600 /etc/NetworkManager/system-connections/eth0.nmconnection
```

### Configure boot files

#### cmdline.txt

Edit `/boot/cmdline.txt`. This must be a single line:

```
console=serial0,115200 console=tty1 root=/dev/mapper/crypted cryptdevice=UUID=<YOUR-LUKS-UUID>:crypted rootfstype=ext4 fsck.repair=yes ip=<YOUR-IP>::<YOUR-GATEWAY>:<YOUR-NETMASK>:<YOUR-HOSTNAME>:<YOUR-INTERFACE>:off rootwait
```

> **Important**: Edit the `cmdline.txt` at the FAT partition root, NOT in any `firmware/` subdirectory. See [Gotchas](#gotchas-and-lessons-learned) below.

#### config.txt

Edit `/boot/config.txt`. Add to the `[all]` section at the end:

```ini
[all]
auto_initramfs=0
initramfs initramfs_2712 followkernel
```

> For Pi 4 or older, use `initramfs initramfs8 followkernel` instead.

### Build initramfs

```bash
ls /lib/modules/
# Pick the 2712 kernel version for Pi 5

update-initramfs -u -k <YOUR-KERNEL-VERSION>
# Example: update-initramfs -u -k 6.12.75+rpt-rpi-2712
```

### Exit chroot

```bash
sync
exit
```

---

## 3. Back on Host — Post-Build

### Copy initramfs to FAT root (CRITICAL)

`update-initramfs` writes to `/boot/firmware/initramfs_2712` inside the chroot. But the Pi firmware reads from the FAT partition root. You **must** copy it:

```bash
sudo cp /mnt/chroot/boot/firmware/initramfs_2712 /mnt/chroot/boot/initramfs_2712
```

> **This is the most commonly missed step.** Without it, the firmware loads the old stock initramfs (~11MB) which has no cryptsetup or dropbear. Your custom one is ~17MB. If your initramfs is suspiciously small, this copy didn't happen.

### Unmount and clean up

```bash
for i in dev proc sys run; do
  sudo umount -l /mnt/chroot/$i 2>/dev/null
done
sudo umount -l /mnt/chroot/boot
sudo umount -l /mnt/chroot
sudo cryptsetup close crypted
sudo umount /mnt/original

sudo kpartx -d pi-base.img
sudo kpartx -d pi-encrypted.img
```

---

## 4. Flash and Boot

### Write to SD card

```bash
sudo dd if=pi-encrypted.img of=/dev/sdX bs=4M status=progress conv=fsync
```

### First boot

1. Insert SD card, connect Ethernet, power on
2. Wait ~15-30 seconds for initramfs to configure networking
3. SSH in:
   ```bash
   ssh root@<YOUR-INITRAMFS-IP>
   ```
4. At the prompt, run:
   ```bash
   cryptroot-unlock
   ```
5. Enter your LUKS passphrase — the Pi continues booting into the full OS

### Resize encrypted partition (first time only)

After the first successful boot into the full OS:

```bash
sudo parted /dev/mmcblk0 resizepart 2 100%
sudo cryptsetup resize crypted
sudo resize2fs /dev/mapper/crypted
sudo reboot
```

---

## 5. SSH Config (Recommended)

The initramfs (dropbear) and full OS (OpenSSH) have different host keys. Configure separate known_hosts to avoid conflicts:

```
Host pi-unlock
    Hostname <YOUR-INITRAMFS-IP>
    User root
    IdentityFile ~/.ssh/<YOUR-KEY>
    UserKnownHostsFile ~/.ssh/known_hosts.initramfs

Host pi
    Hostname <PI-HOSTNAME>.local
    User <YOUR-USER>
    IdentityFile ~/.ssh/<YOUR-KEY>
```

---

## Gotchas and Lessons Learned

### The `/boot` vs `/boot/firmware` confusion

This is the #1 source of errors and wasted hours.

| Context | FAT partition mounted at | `cmdline.txt` firmware reads |
|---------|--------------------------|------------------------------|
| Running Pi | `/boot/firmware/` | `/boot/firmware/cmdline.txt` |
| Chroot (boot mounted at `/mnt/chroot/boot`) | `/mnt/chroot/boot/` | `/mnt/chroot/boot/cmdline.txt` |

When working in the chroot, there will be a `firmware/` **subdirectory** inside your mounted boot partition. This is where `update-initramfs` and the `z50-raspi-firmware` hook write files. But the Pi firmware bootloader reads from the FAT root, not the subdirectory.

**Rules**:
- Edit `cmdline.txt` and `config.txt` at the FAT root (e.g., `/mnt/chroot/boot/cmdline.txt`)
- After `update-initramfs`, copy `firmware/initramfs_2712` up to the FAT root
- If something seems wrong, check file sizes — stock initramfs is ~11MB, custom is ~17MB

### The `ip=` device field

The kernel `ip=` parameter format: `ip=client:server:gateway:netmask:hostname:device:autoconf`

The initramfs `configure_networking()` function in `/scripts/functions` does this:
1. Splits `IP` on `:` delimiters
2. Sets `DEVICE=$6` (the device field)
3. Waits for `/sys/class/net/$DEVICE` to appear

If you put `any` in the device field, it literally sets `DEVICE=any` and waits for `/sys/class/net/any`. That interface will never exist. The initramfs silently times out after 180 seconds and continues without networking.

**Correct usage:**
- For a specific interface: `ip=192.168.1.50::192.168.1.1:255.255.255.0:hostname:eth0:off`
- For any interface with DHCP: `ip=::::hostname::dhcp` (note: two colons = empty device field)

### Initramfs must be copied after every rebuild

Any time `update-initramfs` runs (during initial setup or later on the running Pi), the output goes to the wrong location. Always follow up with a copy to the FAT root.

### Pi 5 network drivers are built-in

The Pi 5 (BCM2712) has the GENET Ethernet driver compiled directly into the kernel — it's not a loadable module. The network interface should appear immediately at boot without any explicit module loading. Listing `genet`, `broadcom`, `pcie_brcmstb` in `/etc/initramfs-tools/modules` is harmless but unnecessary on the 2712 kernel.

### Interface naming

The Pi 5 may use either `eth0` or `end0` depending on the OS version and whether predictable network interface names are enabled. Check which name your system uses before setting static IPs. When in doubt, use an empty device field with DHCP first to discover the name, then switch to static.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| Pi boots but no network / not in router DHCP table | Wrong `ip=` format (check device field), wrong `cmdline.txt` edited, or old initramfs loaded |
| SSH connection refused | Dropbear not in initramfs — check `initramfs_2712` file size (should be ~17MB, not ~11MB) |
| "No such device" during boot | UUID in `crypttab`/`cmdline.txt` doesn't match, or cryptroot hook resolved UUID to loop device path (apply the patch) |
| Timeout waiting for passphrase | Increase `TIMEOUT` in `cryptroot-unlock` |
| Boots to emergency shell | Check `fstab` — root must point to `/dev/mapper/crypted`, not `PARTUUID` |
| Network comes up but SSH hangs | Check `authorized_keys` in `/etc/dropbear/initramfs/` has your public key |

---

## Resources

- [Raspberry Pi Documentation — Boot Configuration](https://www.raspberrypi.com/documentation/computers/configuration.html)
- [initramfs-tools(8)](https://manpages.ubuntu.com/manpages/noble/man8/initramfs-tools.8.html)
- [cryptsetup(8)](https://man7.org/linux/man-pages/man8/cryptsetup.8.html)
- [Kernel ip= parameter documentation](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)

---

## Appendix A: WiFi Unlock (Instead of Ethernet)

If you need to SSH into the initramfs over WiFi instead of (or in addition to) Ethernet, this appendix covers the additional setup required. Everything in the main guide still applies — this section adds WiFi-specific configuration on top.

> **Security warning**: The WiFi PSK is stored in plaintext inside the initramfs on the **unencrypted** boot partition. Anyone with physical access to the SD card can extract your WiFi password. This is an inherent limitation of pre-boot WiFi.

### WiFi hardware overview

Both the Pi 5 and Pi 4 use the same WiFi chip (Infineon CYW43455) over SDIO. The Pi 5's PCIe lane is for NVMe/M.2, not WiFi.

| | RPi 5 | RPi 4 |
|---|---|---|
| WiFi chip | CYW43455 | CYW43455 |
| Bus | SDIO | SDIO |
| Kernel module | `brcmfmac` | `brcmfmac` |
| Extra module | `brcmfmac_wcc` (required) | `brcmfmac_wcc` (needed on Trixie/6.12+) |
| Interface name | `wlan0` | `wlan0` |

### A.1 — Additional kernel modules

Add to `/etc/initramfs-tools/modules` (alongside the existing crypto modules):

```
# WiFi
brcmfmac
brcmutil
brcmfmac_wcc
cfg80211
rfkill
```

### A.2 — Create wpa_supplicant config

Create `/etc/initramfs-tools/wpa_supplicant.conf`:

**WPA2-PSK:**
```conf
ctrl_interface=/tmp/wpa_supplicant
country=US

network={
    ssid="YourNetworkName"
    scan_ssid=1
    psk="YourPassphrase"
    key_mgmt=WPA-PSK
}
```

**WPA3-SAE:**
```conf
ctrl_interface=/tmp/wpa_supplicant
country=US

network={
    ssid="YourNetworkName"
    scan_ssid=1
    psk="YourPassphrase"
    key_mgmt=SAE
    ieee80211w=2
}
```

> Change `country=US` to your regulatory domain. You can use `wpa_passphrase "SSID" "passphrase"` to generate a hashed PSK instead of plaintext, but this is only minor obfuscation.

### A.3 — Create the initramfs hook

Create `/etc/initramfs-tools/hooks/enable-wireless`:

```bash
#!/bin/sh
set -e
PREREQ=""
prereqs() { echo "${PREREQ}"; }
case "${1}" in prereqs) prereqs; exit 0;; esac

. /usr/share/initramfs-tools/hook-functions

# Binaries
copy_exec /sbin/wpa_supplicant
copy_exec /sbin/wpa_cli
copy_exec /sbin/rfkill

# WiFi kernel modules (dependencies pulled automatically)
manual_add_modules brcmfmac
manual_add_modules brcmfmac_wcc

# Firmware — copy all brcmfmac files (covers Pi 4 + Pi 5 + symlink targets)
for f in /lib/firmware/brcm/brcmfmac*; do
    [ -e "$f" ] && copy_file firmware "$f" || true
done
# Cypress directory (symlink targets for the above)
if [ -d /lib/firmware/cypress ]; then
    for f in /lib/firmware/cypress/cyfmac43455*; do
        [ -e "$f" ] && copy_file firmware "$f" || true
    done
fi
# Regulatory database
for f in /lib/firmware/regulatory.db /lib/firmware/regulatory.db.p7s; do
    [ -e "$f" ] && copy_file firmware "$f" || true
done

# Config
copy_file config /etc/initramfs-tools/wpa_supplicant.conf /etc/wpa_supplicant.conf

# RPi brcmfmac modprobe options (roamoff, feature_disable)
if [ -f /lib/modprobe.d/rpi-brcmfmac.conf ]; then
    mkdir -p "${DESTDIR}/lib/modprobe.d"
    cp /lib/modprobe.d/rpi-brcmfmac.conf "${DESTDIR}/lib/modprobe.d/"
fi
```

### A.4 — Create the premount script

Create `/etc/initramfs-tools/scripts/init-premount/a_enable_wireless`:

> The `a_` prefix ensures this runs alphabetically before other init-premount scripts, so WiFi is up before `configure_networking`.

```bash
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in prereqs) prereqs; exit 0;; esac

. /scripts/functions

INTERFACE="wlan0"
WAIT_LIMIT=30
AUTH_LIMIT=60

# Wait for wlan interface to appear (module + firmware loading takes time)
log_begin_msg "Waiting for ${INTERFACE}"
limit=${WAIT_LIMIT}
while [ ! -d "/sys/class/net/${INTERFACE}" ] && [ $limit -gt 0 ]; do
    sleep 1
    limit=$(( limit - 1 ))
done

if [ ! -d "/sys/class/net/${INTERFACE}" ]; then
    log_failure_msg "${INTERFACE} did not appear after ${WAIT_LIMIT}s"
    exit 0  # Don't panic — fall through so user can debug from console
fi
log_success_msg "${INTERFACE} found"

# Unblock WiFi (may be soft-blocked on some setups)
/sbin/rfkill unblock wifi 2>/dev/null || true

# Let firmware finish initializing
sleep 2

# Start wpa_supplicant
log_begin_msg "Starting wpa_supplicant"
/sbin/wpa_supplicant -i${INTERFACE} -c/etc/wpa_supplicant.conf \
    -P/run/initram-wpa_supplicant.pid -B -f /tmp/wpa_supplicant.log

# Wait for WPA association
log_begin_msg "Waiting for WiFi association (max ${AUTH_LIMIT}s)"
limit=${AUTH_LIMIT}
while [ $limit -gt 0 ]; do
    status=$(/sbin/wpa_cli -p/tmp/wpa_supplicant -i${INTERFACE} status 2>/dev/null \
        | grep wpa_state || echo "wpa_state=UNKNOWN")
    if [ "$status" = "wpa_state=COMPLETED" ]; then
        break
    fi
    sleep 1
    limit=$(( limit - 1 ))
done

if [ "$status" = "wpa_state=COMPLETED" ]; then
    log_success_msg "WiFi connected"
else
    log_failure_msg "WiFi association failed after ${AUTH_LIMIT}s"
fi

# Now let configure_networking run (uses kernel ip= parameter on wlan0)
configure_networking
```

### A.5 — Create the cleanup script

Create `/etc/initramfs-tools/scripts/local-bottom/kill_wireless`:

```bash
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in prereqs) prereqs; exit 0;; esac

# Kill wpa_supplicant so the full OS can manage WiFi
if [ -f /run/initram-wpa_supplicant.pid ]; then
    kill "$(cat /run/initram-wpa_supplicant.pid)" 2>/dev/null || true
fi
ip link set wlan0 down 2>/dev/null || true
```

### A.6 — Make everything executable

```bash
chmod +x /etc/initramfs-tools/hooks/enable-wireless
chmod +x /etc/initramfs-tools/scripts/init-premount/a_enable_wireless
chmod +x /etc/initramfs-tools/scripts/local-bottom/kill_wireless
```

### A.7 — Change networking config for WiFi

In `/etc/initramfs-tools/initramfs.conf`, change the device and IP to use `wlan0`:

```bash
DEVICE=wlan0
IP=<YOUR-IP>::<YOUR-GATEWAY>:<YOUR-NETMASK>:<YOUR-HOSTNAME>:wlan0:off
```

Or for DHCP:
```bash
DEVICE=wlan0
IP=:::::wlan0:dhcp
```

In `/boot/cmdline.txt`, update the `ip=` parameter similarly:

```
... ip=<YOUR-IP>::<YOUR-GATEWAY>:<YOUR-NETMASK>:<YOUR-HOSTNAME>:wlan0:off ...
```

Or for DHCP:
```
... ip=:::::wlan0:dhcp ...
```

Then rebuild initramfs and copy to FAT root as described in the main guide.

### WiFi-specific gotchas

**`brcmfmac_wcc` is required on Pi 5** (and Pi 4 on Trixie/kernel 6.12+). Without it, `wlan0` appears but WPA authentication fails silently. This module handles SAE external authentication and firmware event setup.

**Module loading delay**: `wlan0` takes several seconds to appear after `brcmfmac` loads (firmware must be parsed). The premount script polls for up to 30 seconds. If you see timeouts, check that firmware files were correctly copied into the initramfs.

**WiFi association takes time**: After wpa_supplicant starts, the WPA handshake can take 5-15 seconds. The premount script waits up to 60 seconds. If association consistently fails, check `/tmp/wpa_supplicant.log` inside the initramfs (connect a monitor or serial console).

**`MODULES=most` is mandatory**: With `MODULES=dep`, the chroot build environment won't detect WiFi hardware, so `brcmfmac` won't be included. The main guide already sets `MODULES=most`.

**Firmware symlinks**: The files in `/lib/firmware/brcm/` are often symlinks to `/lib/firmware/cypress/`. The hook copies both directories to be safe.

**Pi 5 WiFi chip cannot be software-reset**: Unlike Pi 4, if the CYW43455 enters a bad state on Pi 5, only a full power cycle recovers it. Not usually a problem during normal boot, but relevant during debugging.

---

*Guide written April 2026. If you found this helpful or found additional gotchas, please contribute back.*
