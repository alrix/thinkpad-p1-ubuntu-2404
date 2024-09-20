# Ubuntu 24.04 on a Thinkpad P1 (Gen1)

This repo contains instructions for installing Ubuntu 24.04 on a Lenovo Thinkpad P1 (Gen1). These instructions may work on other generations of P1 and also the X1 Extreme. 

## Motivation

Since I first purchased the laptop in 2019, I have used it with various OS, including CentOS & RHEL, Pop! OS, Windows 10 + WSL2, Arch, OpenSUSE Tumbleweed and Fedora Silverblue. Key drivers for myself are a polished system with good UI but I also prefer stability over latest/greatest.

For most things, I use the ArchLinux wiki - I find this is the better source of information than following random blogs or troubleshooting wiki's.

The motivations for switching to Ubuntu 24.04 include the longevity of support being an LTS release, ZFS support (something I wanted to try out), and the generally good reviews. I have used Ubuntu since its inception, and use it regularly as a server OS in my day-to-day work. The switch to Unity and some of Canonical's methods didn't appeal so I moved away to other desktop OS in recent years. I'm still not a fan of Snap package management, but as with any Linux distro, nothing is perfect and after 20 years of using Linux I'm still looking for that ideal setup...

## Installation Steps

### 1 Ubuntu Setup
Make sure the disk management in the BIOS is set to AHCI. Install Ubuntu 24.04 from USB stick - I chose the basic installation. Use ZFS (with encryption) for hard-disk setup.

### 2 NVIDIA Drivers and setup

All the drivers work out the box with via the install if you select Install Third Party Software.

### 3 Mirrored ZFS Setup

My laptop has two SSDs installed. This gives me the opportunity for a fully mirrored setup.

In summary, the Ubuntu install will partition one of the disks with the following setup (numbers will vary depending on physical disk):

```
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2203647   1.0 GiB     EF00  EFI system partition
   2         2203648         6397951   2.0 GiB     8300  Linux filesystem
   3         6397952        14786559   4.0 GiB     8300  Linux filesystem
   4        14786560       500115455   231.4 GiB   8300  Linux filesystem
```

Part 1 is the EFI mounted under /boot/efi
Part 2 is /boot
Part 3 is SWAP
Part 4 is the ZFS volume

The ZFS volume has two pools:

```
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
bpool  1.88G   220M  1.66G        -         -     0%    11%  1.00x    ONLINE  -
rpool   230G  37.3G   193G        -         -     4%    16%  1.00x    ONLINE  -
```

bpool - the boot pool
rpool - the root pool 

Nice naming!

What we then do is mirror this setup onto the other disk. First step is to identify the current disk then mirror that to the other disk. Identifying the disk - there are many options such as lsblk - you can even use the crypttab to identify the partition, then look it up in /dev:

```
# cat /etc/crypttab

# ls -l /dev/disk/by-partuuid | grep {partuid}
```

Once you know the disk device - along the lines of /dev/nvme0n1 or /dev/nvme1n1, mirror the partition table to the other disk. I used gdisk and created the partitions manually using the start and end sectors. You can dump and restore the table using tools such as gparted.

Now your partition table is mirrored, time to setup the zfs mirrors. Use zpool status to obtain the disk partitions assigned to the current pools. Look up the corresponding disk partition on the other drive under /dev/disk/by-id.

Then mirror the pools using zpool attach. The ArchLinux Wiki has some great instructions for this: https://wiki.archlinux.org/title/ZFS#Attaching_a_device_to_(create)_a_mirror

Watch the status of the pools using zpool status or zpool list -v. You can even detach the old disks and re-add if you want, using new labels. 

Finally, run:
```
update-initramfs -u
```


## 4 SWAP Setup

SWAP uses a LUKS encrypted partition. You have a number of options here - either detach and setup a RAID-1 mirror, then encrypt using LUKS, or simply extend the SWAP on the system by replicating the existing setup on the second disk (I took this approach). 

Check the settings of the swap partition in /etc/crypttab. Identify the corresponding partition in /dev/disk/by-partuuid on the other disk, then follow these instructions to setup the encrypted swap on the second disk - https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption

Make sure your encrypted device is added to /etc/crypttab and use the same password you used for the encryption setup during the install to encrypt this disk. Update /etc/fstab accordingly. 

Finally,run:
```
update-initramfs -u
```

## 5 EFI Setup

I took this from a stackexchange message.  Ubuntu's solution to a redundant ESP is to just to create and mount two of them, and reconfigure grub, instead of creating one on a superblock 1.0 RAID-1.

The name of the second mount point doesn't matter. Since a single ESP is usually mounted under /boot/efi, mounting the second ESP under something like /boot/eficopy would be natural.

Both ESPs have to be mounted automatically via /etc/fstab in case a grub package update happens.

It's important that both ESPs have the right GPT type (i.e. C12A7328-F81F-11D2-BA4B-00A0C93EC93B). Sizing them 200 MiB each is sufficient.

The initial setup then requires reconfiguring grub:

```
dpkg-reconfigure grub-efi-amd64
dpkg-reconfigure shim-signed
dpkg-reconfigure grub-efi-amd64-signed
update-initramfs -u
```

### 6 Final Tuning

I add the icc profile in this repo to improve the screen colours. 


