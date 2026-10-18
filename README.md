# Custom Arch Linux Setup (LUKS + Btrfs + i3)

A minimal, secure, and keyboard-driven Arch Linux installation configuration built from bare metal. I tried to explain as much as I could everything. If there is a $ sign is a command or a value that needs to be put in configuration files, -> are just plain and simple explanations. Goes without saying that to start this installation process a USB with a Arch Linux image is needed. This build gives you the opportunity to make any machine lightning fast, minimum requirements are electricity, internet connection and patience. NO minimum hardware requirements!

---

## Tech Stack & Features
* **OS:** Arch Linux (Rolling release)
* **Bootloader:** GRUB (with encrypted disk support)
* **Filesystem:** Btrfs (with subvolumes '@' and '@home', compressed with 'zstd')
* **Encryption:** LUKS (Full-disk encryption)
* **Window Manager:** i3-wm (Tiling)
* **Terminal Emulator:** Alacritty
* **Shell:** Bash
* **Audio:** Pipewire + WirePlumber

---

## Step-by-Step Installation Process

### 1. Connect to internet
* **$ iwctl -> To enter wifi management utilities;
* **$ device list -> To see available wifi interfaces;
* **$ station [wifi interface] get-networks -> To see available wifi networks;
* **$ station [wifi interface] connect [wifi network] -> To connect;
* **$ exit -> To exit wifi management utilities;
### 2. Check internet connection
* **$ ip addr show -> To see IP & if up/down;
* **$ ping -c 3 archlinux.org -> To check packet transmission;
### 2.1 (Optional) SSH connection
On the current machine:
* **$ passwd -> To set password for root account installer;
On other machine:
* **$ ssh root@[ip address] -> To connect remotely;
### 3. Partitioning the Disk (NVMe)
* **$ lsblk -> To check associated disks;
* **$ fdisk /dev/nvme0n1 -> To start partitioning through fdisk console;
* **$ g -> To give new partition layout to disk;
* **$ n -> Creates a new partition -> $ +512M -> size needed;
* **$ t -> Changes partition type, select partition 1 and type 1 for EFI system;
* **$ n -> Creaes new partition -> Remainder size;
* **$ w -> Writes partitions to disk;
* **$ exit -> Exits fdisk console;
### 3.1 Format partitions
* **$ mkfs.fat -F32 /dev/nvme0n1p1 -> Formats EFI partition as FAT32;
* **$ cryptsetup luksformat /dev/nvme0n1p2 -> Encrypts main partition with LUKS;
### 3.2 Open encrypted container
* **$ cryptsetup open /dev/nvme0n1p2 cryptroot -> Maps unlocked drive to /dev/mapper/cryptroot;
### 3.3 Format unlocked container as btrfs
* **$ mkfs.btrfs /dev/mapper/cryptroot
### 3.4 Create btrfs subvolumes by mounting raw btrfs filesystem temporarily (@ for root, @home for personal files)
* **$ mount -o subvol=@,noatime,compress=zstd /dev/mapper/cryptroot /mnt
* **$ mkdir -p /mnt/home
* **$ mount -o subvol=@home,noatime,compress=zstd /dev/mapper/cryptroot /mnt/home
* **$ mkdir -p /mnt/boot
* **$ mount /dev/nvme0n1p1 /mnt/boot
### 3.5 Install Arch Linux kernel by using script pacstrap that installs packages to the specified new root directory, /mnt is the target mount point where new Arch file system (btrfs subvolumes) is mounted, base is the foundational package group for an Arch system (core utilities, pacman, etc.), linux is the actual kernel and its modules, linux-firmware is the essential firmware files for hardware drivers (wifi, graphics, etc.), btrfs-progs are the utilities needed to manage and work with the btrfs filesystem.
* **$ pacstrap /mnt base linux linux-firmware btrfs-progs
### 3.6 Generate the filesystem table (fstab)
* **$ genfstab -U /mnt >> /mnt/etc/fstab
### 3.7 Check the file generated
* **$ cat /mnt/etc/fstab
### 3.8 Chroot into the new system, since we are still running inside the USB live installation, let's enter the fresh installed system as root.
* **$ arch-chroot /mnt
### 3.9 Set timezone
* **$ ln -sf /usr/share/zoneinfo/[Continent]/[City] /etc/localtime
* **$ hwclock --systohc
### 4. Uncomment language in locale.gen
* **$ echo "[language and locale chosen e.g en_US.UTF-8 UTF-8]" >> /etc/locale.gen
* **$ locale-gen
* **$ echo "[language and locale chosen e.g LANG=en_US.UTF-8]" > /etc/locale.conf
### 5. Give your machine a name (hostname)
* **$ echo "[hostname]" > /etc/hostname
### 6. Set root password
* **$ passwd
### 7. Configure the initramfs (crucial for encryption!) since the root drive is encrypted with LUKS, the linux kernel needs to know how to decrypt it before it tries to mount the filesystem, let's open th mkinitcpio configuration file, find the HOOKS="" line and change it, save, exit and regenerate the initramfs image. Do not forget in next steps to change the MODULES line as well and uncomment line to enable LUKS!
* **$ pacman -S nano
* **$ nano /etc/mkinitcpio.conf -> $MODULES=(nvme btrfs) $HOOKS=(base udev autodetect microcode modconf block encrypt filesystems fsck);
* **$ mkinitcpio -P -> Rebuilds initramfs image
### 7.1 Install bootloader GRUB
* **$ pacman -S grub efibootmgr
* **$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
### 7.2 Tell GRUB where encrypted partition lives using its UUID
* **$ blkid /dev/nvme0n1p2 -> Finds the encrypted partition's UUID;
* **$ nano /etc/default/grub -> Boot parameters: enable GRUB_ENABLE_CRYPTODISK=y, configure kernel parameter to decrypt disk at boot using LUKS partition UUID: $ GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=[YOUR_UUID]:cryptroot root=/dev/mapper/cryptroot"
### 7.3 Generate GRUB configuration file
* **$ grub-mkconfig -o /boot/grub/grub.cfg
### 7.4 (Troubleshooter optional step: If chroot is missing internet)
* **$ exit
* **$ cp /etc/resolv.conf /mnt/etc/resolv.conf
### 7.5 (Troubleshooter optional step: Chroot back into the system)
* **$ arch-chroot /mnt
### 8 Install network manager & iwd
* **$ pacman -S networkmanager iwd
### 8.1 Enable network manager
* **$ systemctl enable NetworkManager
* **$ exit
* **$ umount -R /mnt
* **$ reboot
### 8.2 Connect wifi through nmcli
* **$ nmcli device wifi list
* **$ nmcli device wifi rescan
* **$ nmcli device wifi connect "[SSID]" password "[PASSWORD]"
* **$ nmcli -a g -> To check status;
### 8.3 To fix mirrorlist use reflector
* **$ pacman -S reflector
* **$ reflector --country [your_country] --protocol https --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
### 8.4 Install i3 & essential utilities
* **$ pacman -S i3-wm i3status i3lock dmenu alacritty xorg xorg-xinit sudo pipewire wireplumber
### 8.5 Create non-root user
* **$ useradd -m -G sudo -s /bin/bash [username]
* **$ passwd [username]
### 8.6 Give user sudo privileges by uncommenting the '%sudo ALL=(ALL:ALL) ALL' line through $EDITOR=nano visudo
### 8.7 To make xorg automatically launch i3 when you type 'startx', add this line to your user's home profile
* **$ echo "exec i3" >> ~/.xinitrc
* **$ startx
