```
Lenovo (ThinkPad) Yoga 11e 6th Gen

Artix Linux-Zen

Intel® Core™ m3-8100Y
11.6" HD IPS TS with Corning® Gorilla® Glass
2W x2 Stereo Dolby® Advanced Audio
```

## 1. Pre-installation

### 1.1 Acquire Artix installation image

Choose `#! base` stable ISO image on Artix Linux [Download](https://artixlinux.org/download.php#official) page depending on your preferred init system. In this page, OpenRC will be used as default.

### 1.2 Prepare an installation media

It is suggested to use [Ventoy](https://www.ventoy.net/en/doc_news.html) if you **already** have hybrid installation media (with many DATA included) such as External SSDs. Use [balenaEtcher](https://etcher-docs.balena.io/) or similiar tool otherwise.

### 1.3 Boot the live environment

> Ensure to **disable** Secure Boot on firmware settings. In this page, UEFI system is used.

Open boot menu by pressing a key during POST and point to your installation media, or set the firmware's boot priority/order. Refer to your Vendor for details. On GRUB menu, optionally configure according to your needs and select "From CD/DVD/ISO: artix.x86_64". Login as root.

### 1.4 Connect to the internet

1. Ensure the network interface is known.

```sh
ip link
```

2. For wireless connection, WLAN card shall not blocked and enabled.

```sh
rfkill unblock wlan
```
```sh
ip link set wlan0 up
```

3. Plug the ethernet cable, or authenticate to wireless network.

```bash
wpa_supplicant -B -i wlan0 -c <(wpa_passphrase 'SSID' 'WPA2-PSK_PASSWORD')
```

4. Configure your network connection, DHCP is preferred.

```sh
dhcpcd
```

5. Verify the internet connection, usually with ping.

```sh
ping artixlinux.org
```

### 1.5 Update the system clock

Activate the NTP daemon to synchronize the computer's real-time clock.

```sh
rc-service ntpd start
```

### 1.6 Partition the disks

Linux kernel uses block device such as `/dev/sda` for SATA and USB flash drive, `/dev/nvme0n1` for NVMe, and `/dev/mmcblk0` for MMC/eMMC. To identify these devices, use lsblk or fdisk.

```sh
lsblk -f
```
```sh
fdisk -l
```

> If your NVMe devices not detected, you may try **permanently disable** Intel VMD on firmware settings. This means, RAID mode is not allowed. In this page, NVMe SSD is used.

The following partition scenario are **needed** for a chosen device:
- Single partition for the root volume `/`, and
- EFI system partition (ESP) to store kernel and initial ramdisk.

Use a partitioning tool like cfdisk to modify partition table.

```sh
cfdisk /dev/nvme0n1
```

#### 1.6.1 Partition table

| Mount point | Partition | Type | Suggested size |
|:---|:---|:---|:---|
| `/` | `/dev/nvme0n1p2` | Linux filesystem | At least 20GB |
| `/boot/` | `/dev/nvme0n1p1` | EFI System | Over 100MB |

### 1.7 Format the partitions

Once the partitions have been created, each newly created partition must be formatted with an appropriate file system. Flash-Friendly File System (F2FS) is suggested format as uses SSD.

```sh
mkfs.f2fs -l 'Artix Linux' -O extra_attr,inode_checksum,flexible_inline_xattr,inode_crtime,verity,sb_checksum,compression /dev/nvme0n1p2
```

If you created an EFI system partition, format it to FAT32.

> ESP must not be formatted if you **already** use other OS such as dual-booting Microsoft Windows. Resizing ESP on GPT system as suggested size requires advanced effort. 

```sh
mkfs.fat -F 32 /dev/nvme0n1p1
```

### 1.8 Mount the file systems

Mount the root volume to `/mnt/`.

```sh
mount -o rw,noatime,gc_merge,inline_xattr,flush_merge,data_flush,checkpoint_merge,compress_algorithm=lz4,compress_extension=*,compress_chksum,compress_cache,atgc,age_extent_cache /dev/nvme0n1p2 /mnt/
```

Mount the ESP to `/mnt/boot/`.

```sh
mount --mkdir -o noatime /dev/nvme0n1p1 /mnt/boot/
```

## 2. Installation

### 2.1 Base system

Use basestrap to install the base, base-devel package groups, and your preferred init system.

```sh
basestrap /mnt/ base base-devel openrc elogind-openrc
```

### 2.2 Linux kernel

There are three kernels Artix provides: linux, linux-lts, and linux-zen. In this page, linux-zen is suggested for balanced optimization and linux-firmware for compability fixes.

```sh
basetrap /mnt/ linux-zen linux-firmware
```

## 3. Configure the system

### 3.1 Fstab

Generate a fstab file.

```sh
fstabgen -U /mnt/ >> /mnt/etc/fstab
```

As uses GPT system, partitions should be mounted persistently by PARTUUID. Use blkid to generate identifiers and append to fstab file.

```sh
blkid >> /mnt/etc/fstab
```

Check the resulting `/mnt/etc/fstab` file, and edit it in case of errors. 

```sh
$EDITOR /mnt/etc/fstab
``` 

#### 3.1.1 File system table

Due to F2FS boot problems with some options, remove `flush_merge,compress_cache,atgc,age_extent_cache` from fstab file. The `/mnt/etc/fstab` file should looks like this.

```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/nvme0n1p2 LABEL=Artix\134x20Linux
PARTUUID=a8d18ea3-a488-4362-bbe1-92f6e821689d	/         	f2fs      	rw,noatime,gc_merge,inline_xattr,data_flush,checkpoint_merge,compress_algorithm=lz4,compress_extension=*,compress_chksum	0 1

# /dev/nvme0n1p1
PARTUUID=166dd973-e7d1-4890-be94-48a93c1b719b      	/boot     	vfat      	rw,noatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro	0 2
```

### 3.2 Chroot

Change root into the new Artix system.

```sh
artix-chroot /mnt/
```

<br>...<br>

```
Compiled and written by owl4ce.
© 2025 owl4ce
https://github.com/owl4ce/exquisitely
```
