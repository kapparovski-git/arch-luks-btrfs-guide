# Custom Arch Linux Setup (LUKS + Btrfs + i3)

A minimal, secure, and keyboard-driven Arch Linux installation built from bare metal. This guide walks through configuring full-disk encryption via LUKS, modern snapshot-ready Btrfs subvolumes with transparent `zstd` compression, GRUB over UEFI, and a lightweight `i3-wm` tiling environment.

---

<img width="1920" height="1062" alt="image" src="https://github.com/user-attachments/assets/f5fb2907-dc5f-441c-80b0-b251439fb608" />


---

## Architecture & Features

* **Firmware:** UEFI with GRUB bootloader
* **Filesystem:** Btrfs (with subvolumes '@' and '@home', compressed with 'zstd')
* **Encryption:** LUKS (Linux Unified Key Setup) full-disk encryption on root
* **Window Manager:** i3-wm (tiling)
* **Terminal Emulator:** Alacritty
* **Audio:** Pipewire + WirePlumber
* **Network Stack:** NetworkManager + iwd

---

## Prerequisites

Before starting, ensure you have:
1. **Target Hardware:** A 64-bit x86_64 machine with UEFI support (Secure Boot disabled in BIOS/firmware settings).
2. **Installation Media:** A bootable USB flash drivr with the latest official [Arch Linux ISO](https://archlinux.org/download/).
3. **Internet Access:** Active Ethernet or Wi-Fi available at the installation site.
4. **Data Backup:** This installation completely overwrites the target disk. Back up any essential data before proceeding.

> **Warning:** This guide assumes your target drive is `/dev/nvme0n1`. If your machine uses a SATA SSD (`/dev/sda`) or a different drive path, change the disk identifiers accordingly before running any disk commands.

## Step-by-Step Installation Process

### 1. Connect to internet
Launch the interactive Wi-Fi shell:
```bash
iwctl
```

Inside the `[iwd]#` prompt, locate your Wi-Fi interface and connect it to your Wi-Fi network:
```bash
device list
station <interface> get-networks
station <interface> connect "<SSID>"
exit
```
### 2. Check internet connection
Check IP, connection status and packet transmission:
```bash
ip addr show
ping -c 3 archlinux.org
```
### 2.1 (Optional) SSH connection
Set root password on the current machine:
```bash
passwd
```
Connect remotely from other machine:
```bash
ssh root@<ip_address>
```
### 3. Disk Partitioning Layout
First let's have a look at how we should partition the chosen disk:

| PARTITION | TYPE | SIZE | MOUNT / PURPOSE
| :--- | :--- | :--- | :--- |
| `/dev/nvme0n1p1` | EFI System (`vfat`) | 512MiB | `/boot` |
| `/dev/nvme0n1p2` | Linux filesystem (LUKS) | Remainder | Root LUKS Container |

To find out which disk we are going to work with:
```bash
lsblk
```
Launch the interactive disk partitioner shell onto the chosen disk:
```bash
fdisk /dev/<disk>
```

Inside the `fdisk`, enter the following sequence to create the GPT table and partitions:
 
```text
g           -> Create a new GPT partition table

n           -> Create partition 1 (EFI)
<Enter>     -> Default partition number (1)
<Enter>     -> Default first sector
+512M       -> Size for EFI partition

t           -> Change partition type
1           -> Select "EFI System"

n           -> Create partition 2 (Root LUKS)
<Enter>     -> Default partition number (2)
<Enter>     -> Default first sector
<Enter>     -> Press Enter to use remaining disk space

w           -> Write changes and exit fdisk automatically
```
### 3.1 Format partitions & create btrfs subvolumes by mounting raw btrfs filesystem temporarily (@ forroot, @home for personal files)

```bash
# Format EFI partition as FAT32
mkfs.fat -F32 /dev/nvme0n1p1

# Encrypts main partition with LUKS
cryptsetup luksFormat /dev/nvme0n1p2

# Opens encrypted container
cryptsetup open /dev/nvme0n1p2 cryptroot

# Format unlocked container as btrfs
mkfs.btrfs /dev/mapper/cryptroot

# Mount encrypted partition onto /mnt
mount /dev/mapper/cryptroot /mnt

# Create subvolumes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home

# Unmount /mnt
umount /mnt

# Remount cleanly with subvolumes and compression, create /home and /boot directories
mount -o subvol=@,noatime,compress=zstd /dev/mapper/cryptroot /mnt
mkdir -p /mnt/home
mount -o subvol=@home,noatime,compress=zstd /dev/mapper/cryptroot /mnt/home
mkdir -p /mnt/boot

# Mount /boot
mount /dev/nvme0n1p1 /mnt/boot
```

### 4. Install Base System & Firmware

Install the essential base packages, kernel, filesystem tools, networking, and microcode:

```bash
# Replace <ucode-package> with intel-ucode (for Intel CPUs) or amd-ucode (for AMD CPUs)
pacstrap /mnt base linux linux-firmware btrfs-progs nano networkmanager <ucode-package>
```

### 5. Generate Fstab & Enter System

Generate the filesystem table using UUIDs and enter the new installation:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

### 6. Set timezone, language, hostname & root password

```bash
# Replace <continent> and <city> accordingly
ln -sf /usr/share/zoneinfo/<continent>/<city> /etc/localtime
hwclock --systohc

# Replace <locale> ( e.g. "en_US.UTF-8 UTF-8" ) and <language> ( e.g. "LANG=en_US.UTF-8" ) accordingly
echo "<locale>" >> /etc/locale.gen
locale-gen
echo "<language>" > /etc/locale.conf

# Replace <hostname> with desired machine name
echo "<hostname>" > /etc/hostname

# Set root password
passwd
```

### 7. Configure the Initramfs for Encryption

Open the initramfs configuration file:
```bash
nano /etc/mkinitcpio.conf
```

Set the modules and hooks to support LUKS and Btrfs:
```text
MODULES=(btrfs)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems)
```

Regenerate the initramfs image:
```bash
mkinitcpio -P
```

### 7.1 Install & Configure GRUB

Install the bootloader packages:
```bash
pacman -S --noconfirm grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

Find the UUID of your encrypted partition (`/dev/nvme0n1p2`):
```bash
blkid -s UUID -o value /dev/nvme0n1p2
```

Edit `/etc/default/grub`:
```bash
nano /etc/default/grub
```

Set the kernel command line parameters with your partition's UUID:
```text
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=<YOUR_LUKS_UUID>:cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@"
```

Generate the boot configuration:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
### 8.1 Enable Networking

Enable NetworkManager so the sysem automatically handles internet connections on boot:
```bash
systemctl enable NetworkManager
```

### 8.2 User Configuration & Sudo Privileges

Create a standard user account and assign it to the administrative `wheel` group:
```bash
useradd -m -G wheel -s /bin/bash <username>
passwd <username>
```

Grant sudo privileges to the `wheel` group:
```bash
EDITOR=nano visudo
```
> Find and uncomment the following line:
> ```text
> %wheel ALL=(ALL:ALL) ALL
> ```

### 8.3 Install Graphical Environment & Utilities

Install the X display server, i3 window manager, terminal, and audio stack:
```bash
echo "exec i3" > /home/<username>/.xinitrc
chown <username>:<username> /home/<username>/.xinitrc
```

### 8.4 Exit Chroot and Reboot

Exit the chroot environment, unmount all partitions, and reboot into your new installation:
```bash
exit
umount -R /mnt
reboot
```
### 8.5 First Boot & Launching the Desktop

1. Remove the USB installation drive.
2. At the GRUB boot menu, select Arch Linux and enter your LUKS passphrase when prompted.
3. Log in using your **new non-root username** and password (do not log in as root).
4. Launch the i3 window manager:

```bash
startx
```

> **Note:** The first time i3 launches, it will prompt you to generate a config file. Press **Enter** to accept the defaults, and choose your preferred modifier key (`Super` or `Alt`).

### 8.6 (Optional) Auto-start X on Login

To launch i3 automatically without having to type `startx`, append this snippet to `/home/<username>/.bash_profile`:

```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
    exec startx
fi
```

### 9. Post-Installation: Automated Snapshots & Rollback Protection

Because this setup uses a Btrfs filesysem, you can create instant system snapshots using Copy-on-Write (CoW). Snapshots take zero additional disk space when created-storage is only consumed as files are modified or deleted over time.

We use **Snapper** paired with **snap-pac** so that Arch automatically captures a snapshot before and after every single `pacman` transacttion. If a rolling update breaks your system, you can inspect changes and restore files within seconds.

---

### 1. Install Snapper & Pacman Hook

Install Snapper along with `snap-pac`:

```bash
sudo pacman -S snapper snap-pac
```

---

### 2. Configure Snapper for the Root Filesystem

Create a Snapper configuration profile targeting your root (`/`) subvolume:

```bash
sudo snapper -c root create-config /
```

Verify that the `root` profile was generated:

```bash
sudo snapper list-configs
```

---

### 3. Configure Snapshots Retention & Cleanup

By default, Snapper keeps an excessive number of hourly and daily snapshots. To prevent your drive fromfilling up quietly, adjust the timeline retention settings.

Edit `/etc/snapper/configs/root`:

```bash
sudo nano /etc/snapper/configs/root
```

Adjust the following limits to lean defaults:

```text
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="2"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

Enable and start the background cleanup timers:

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

---

### 4. How It Works: Hands-on Usage

#### Automatic Snapshots on Update
You do not need to trigger backups manually for updates. Whenever you run:

```bash
sudo pacman -Syu
```

`snap-pac` automatically creates a `pre` snapshot (snapshot state right before packages install) and a `post` snapshot (state right after completion).

#### View All Saved Snapshots
List existing snapshots and their IDs:

```bash
sudo snapper -c root list
```

#### Inspect Changes Between Snapshots (Diffing)
To see every file added (`+`), modified (`c`), or deleted (`-`) between snapshot 1 and your current live system (`0`):

```bash
sudo snapper -c root status 1..0
```

To see the exact code or config diff of a specific file:

```bash
sudo snapper -c root diff 1..0 /etc/fstab
```

#### Recovering a Broken File
All snapshots reside in `/.snapshots/<ID>/snapshot/`. If a configuration file gets corrupted, copy the working version directly out of the desired snapshot:

```bash
sudo cp /.snapshots/<SNAPSHOT_ID>/snapshot/etc/<config_file> /etc/<config_file>
```
## Window Manager & Environment

The installation sets up basic `i3` and `alacritty` packages. For the exact keybindings, status bar configs, and color schemes used on this system, see my [dotfiles repository](https://github.com/kapparovski-git/dotfiles).

LFG!
