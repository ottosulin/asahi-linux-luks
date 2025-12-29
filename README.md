# Asahi Linux LUKS guide

Adapted from David Alger's blog post what worked in Dec 2025 on my Macbook Air

https://davidalger.com/posts/fedora-asahi-remix-on-apple-silicon-with-luks-encryption/

## Build Asahi Fedora USB

```
dnf install arch-install-scripts bubblewrap dosfstools e2fsprogs gdisk mkosi openssl pandoc rsync systemd-container
python3 -m pip install --user git+https://github.com/systemd/mkosi.git@v23
git clone https://github.com/leifliddy/asahi-fedora-usb
cd asahi-fedora-usb
```

Before running the following command check with lsblk the correct device
`./build.sh -d /dev/sda`

## U-Boot
```
# press down any button during boot to interrupt default boot script
bootmenu
usb start
# Check device number, most likely Device 0
usb storage 
# The following command should show file named "bootaa64.efi"
ls usb 0:1 efi/boot/ 
# Load the EFI file into memory
load usb 0:1 ${kernel_addr_r} efi/boot/bootaa64.efi
# Boot it as an EFI application
bootefi ${kernel_addr_r}
```

## Configure LUKS

```
# Identify the root filesystem partition,
# in the example below partition labeled "fedora_asahi" i.e. "nvme0n1p6":
[root@fedora ~]# lsblk -f /dev/nvme0n1
NAME        FSTYPE FSVER LABEL        UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1
├─nvme0n1p1 apfs                      4ccf344c-1842-4ed2-98f7-d34a509f5a88
├─nvme0n1p2 apfs                      dbb4789e-c51d-46bf-8332-90a43b4e4fa7
├─nvme0n1p3 apfs                      b98ec259-629b-4aee-9f26-02c5098abcee
├─nvme0n1p4 vfat   FAT32 EFI-FEDORA   B01E-2641                             419.8M    16% /run/.system-efi
├─nvme0n1p5 ext4   1.0   fedora_boot  5b094e58-d15f-4be2-85ff-147859c7b118
├─nvme0n1p6 btrfs        fedora_asahi dd08a2bf-ae63-44e1-881d-fbb8928af4fb
└─nvme0n1p7 apfs                      b465c845-eaef-4bcb-aac9-865c42260844
```

Everything below assumes the partition is `nvme0n1p6`.

```
# Shrink the btrfs filesystem to make room for the LUKS header. 
# Recommended minimum is 32 MiB, twice the size of a default LUKS 2 header:
mount /dev/nvme0n1p6 /mnt
btrfs filesystem resize -32M /mnt
umount /dev/nvme0n1p6

# LUKS encrypt the root filesystem partition in-place.
# This will destroy everything on the partition, please be careful!
# This may take from minutes to an hour depending on partition size and hardware
cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/nvme0n1p6

# Open LUKS partition and mount root and home filesystems
cryptsetup open /dev/nvme0n1p6 fedora-root
cryptsetup status fedora-root
mount -o subvol=root /dev/mapper/fedora-root /mnt
mount -o subvol=home /dev/mapper/fedora-root /mnt/home

# Mount boot and efi filesystems which should be 
# the two partitions immediately preceding the one encrypted with LUKS.
# For me mounting the efi filesystem failed on first try but 
# worked when I just rerun the command.
mount /dev/nvme0n1p5 /mnt/boot
mount /dev/nvme0n1p4 /mnt/boot/efi

# Store LUKS UUID for next part
export LUKS_UUID=$(cryptsetup luksUUID /dev/nvme0n1p6 | tee /dev/stderr)
```

## GRUB

```
# Enter chroot
arch-chroot /mnt /bin/bash

# Set up crypttab and GRUB
touch /etc/crypttab
chmod 0600 /etc/crypttab
echo "fedora-root UUID=${LUKS_UUID} none" >> /etc/crypttab
cat /etc/crypttab
perl -i -pe 's/(GRUB_CMDLINE_LINUX_DEFAULT)="(.*)"/$1="$2 rd.luks.uuid='"${LUKS_UUID}"'"/' /etc/default/grub
cat /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut -f

# If dracut -f gives out an error check the kernel version:
ls /lib/modules
# If there are multiple versions you'll most likely want to use 
# the newer kernel version with a command like the following:
dracut -f --kver 6.17.12-400.asahi.fc42.aarch64+16k

# After shutdown and before reboot remove the USB drive
shutdown
```
