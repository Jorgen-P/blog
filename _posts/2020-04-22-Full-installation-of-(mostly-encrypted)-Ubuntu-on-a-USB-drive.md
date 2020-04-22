---
layout: post
title: "Full installation of (encrypted) Ubuntu to USB drive"
date: 2020-04-22 00:00:00 +0000
---

# Introduction
I have often found live Linux USB-drives useful in diagnosing and fixing damaged operating system installations. While it is possible to use a regular Ubuntu or other Linux distribution live CD or USB, I find that I often go back and forth between the system I am attempting to repair and the live USB. This leads to hassle with wireless networks and setting my keyboard to my usual Swedish layout, having to repeat this every boot. Performing a full installation of Ubuntu on a USB-drive enables me to keep my settings, Wi-Fi-passwords, open browser tabs etc. between reboots. Another benefit is that it is simple to encrypt the system the same way as a regular installation (very good if one accidentally leaves the USB-drive in a computer somewhere). Additional use cases I can think of include having a persistent encrypted system for privacy if using a shared computer or some more consistency if using computers that one does not have control over.

I generally avoid installing proprietary graphics drivers if the goal is to have the system be very portable but if you mostly use it for your computer with an Nvidia graphics card it might be worth installing its driver.

I also use (mostly) this method of installation when doing a more custom install than the Ubuntu installer makes simple, such as when installing to bcache or similar. Another benefit is that this method of installation can be used from any running Linux system. For example it is possible to use the resulting USB to install Ubuntu to another machine.

If you want to install another version of Ubuntu replace all occurrences of *bionic* with the appropriate name. If using Ubuntu I recommend using only LTS versions for this because of their longer supported lifetime compared to other releases and the probably low frequency of use for this kind of USB-drive.

# Variations
We will be installing Ubuntu from an existing Linux environment. If you do not require the ability to boot under both UEFI and legacy modes you can try installing using the regular Ubuntu installer (on live USB or CD) to the USB-drive. Make sure to select the USB-drive as the target when the installer asks where you would like to install both the main system and GRUB since installing to the wrong target will overwrite data there.

If your USB is only meant for older computers you can use an MBR partition table and only install grub for the i386-pc target, in this case install the grub-pc package and skip `grub-efi-amd64`. You will not need the bios_grub or EFI partitions.

# Requirements
* Some Linux experience
* A USB-drive of at least 8 GB (more is nice but not required)
* An existing Linux environment (can be a live USB or CD)

If you are new to Linux I recommend starting out using it in a virtual machine (VM). Virtualbox is easy to use for this purpose. That is how I started and is still how I do most of my experimentation. The chief reason is that changing partitions and file systems outside a virtual machine can destroy data so you should have a decent idea of what you are doing. Trying things out in a VM is a great way to get experience without risking your real system.

To get more information about packages you can use `apt show <package-name>` for a description of a package or man <command>, e.g `man mkfs`.

Generally <> signs are used in this guide to indicate that another value should be substituted, such as your username or disk.

# Overview

1. Select a suitable USB-drive
  - Ideally a USB3 drive for speed (optional)
  - At least 16 GB in size for a comfortable experience but if you are careful about using storage 8 GB or possibly less works.
1. Create partitions and file systems on USB-drive
1. Install Ubuntu using deboostrap
1. Chroot into the new Ubuntu system
  * Install necessary packages
  * Modify configuration files
1. Boot the new system

`/mnt` will be used as the mount point of the USB-drive.

# Setup

## Creating partitions and file systems

- Consider the overall partitioning scheme and whether LVM or encryption will be used
  * I will not be using LVM
- Create a GRUB/BIOS partition 1 MiB in size
- Create an EFI partition with a FAT file system
- Create an optionally encrypted partition for the root file system
  - Use most of the space for this
  - If not using encryption, create an EXT4 file system here in Gparted
    - Otherwise create it as cleared for now
  - If using encryption a separate /boot partition makes things easier
    - Make this at least a few hundred MiB but a GiB is convenient if there is space for it.
    - Use EXT4 file system

Using a tool such as Gparted makes it easier to get a nice overview of the current layout of a disk. **Make sure to select the correct drive in the top right hand corner**.
Gparted can create a partition table under `Device/create partition table`.
If we enable `view/Device Information` in Gparted, a little box appears which tells us among other things the current type of partition table, usually this will be MBR which is called msdos by Gparted. We will be using GPT because it has some advantages such as being able to handle more partitions without using tricks with logical partitions and increased resistance to corruption. If you need your USB to work with older computers that can not handle GPT you can try using MBR but I have not tested it.

I am unsure of the minimum size for the EFI system partition, using FAT16 may allow it to be quite small as mine is almost unused. Even on my PC which dual boots Linux Mint and Windows 10 only 32 MiB is used. The [arch wiki](https://wiki.archlinux.org/index.php/EFI_system_partition) recommends a larger size but only to allow for other operating systems, since the USB-drive uses only Linux this should not be an issue.

Format the bios_grub partition as cleared in Gparted, this removes any old traces of a file system.

![](/assets/gparted-usb-devinfo1.png)

Set the bios_grub flag on the bios-grub partition by right clicking the partition and selecting manage flags. On my version of Gparted this required applying all changes first. Also set the boot and ESP (*EFI system partition* I believe on the EFI partition).

If encryption will be used use `sudo cryptsetup luksFormat /dev/sd<Xn>` where Xn is the partition to use for LUKS so in my case `sudo cryptsetup luksFormat /dev/sdc4`. Do not forget the password you select here or the contents of the encrypted file system will become permanently unavailable.

If using encryption we need to unlock the encrypted partition before creating its file system. Use `sudo cryptsetup open /dev/<partition> <name-of-unlocked-device>` so `/dev/sdc4` and *mobile-crypt* here. For the name you can select pretty much whatever you want. After unlocking we can create the file system using `mkfs -t ext4 /dev/mapper/<name-of-unlocked-device>`.


## Mounting file systems
At this point we mount everything under `/mnt` the way we want it to be in the new system.

![Image of USB-partitions](/assets/usb-mounted.png)

*Output of `lsblk /dev/sdc` showing how everything is mounted. The columns of most interest here are NAME, SIZE and MOUNTPOINT.*

We must create directories to mount on. After mounting the root file system on `/mnt` (`mount /dev/<root-partition> /mnt` or `/dev/mapper/<name-of-unlocked-device> /mnt` if using encryption) we create the boot directory there for mounting the /boot file system if using one of those.

## Installation
`sudo deboostrap --arch=amd64 --components=main,universe bionic /mnt`

This will install the 64-bit (x86-64) version of Ubuntu Bionic (18.04) into /mnt and enable universe packages. If you only want to use the most well supported packages remove universe and use only main, there are additional repositories (multiverse, restricted) but you should understand their terms before enabling them. This step can take a little while, particularly if you have a lower speed internet connection or USB-drive.

## Editing configuration files and installing packages
### File systems, crypttab and fstab
Now we need to create the fstab and crypttab (if using encryption) files in the /etc directory of the USB-drive.

To find the uuid of the LUKS partition we can use `sudo blkid /dev/sd<Xn>` on the same partition we used luksFormat on earlier.
To verify that we entered the value correctly we can use `ls /dev/disk/by-partuuid/<uuid>` to check that the path exists.

This is my `/etc/crypttab` on the USB-drive:
```
mobile-crypt /dev/disk/by-uuid/98a7f22c-4110-43c9-9a12-9715285c6704 none luks
```

This is my `/etc/fstab` on the USB-drive:
```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# root
/dev/mapper/mobile-crypt / ext4 defaults 0 1
# boot
UUID=f2d6c425-f07f-4171-b056-b6aa1643d367 /boot ext4 defaults 0 2
# EFI
PARTUUID=0c2ede7f-fe9d-4513-9cd2-d16866c96e71 /boot/efi vfat umask=0077 0 2
```
We do not have to use UUID or PARTUUID but it is preferable since the /dev/sdX name of the drive can change, particularly when drives are removed or added, and a label risks collision with a label on a disk in the machine we insert the USB-drive into.

## Chroot

To prevent some errors first run this outside the chroot environment:

```
sudo mount --bind /dev/ /mnt/dev/;
sudo mount --bind /tmp/ /mnt/tmp/;
sudo mount --bind /proc/ /mnt/proc/;
sudo mount --bind /sys/ /mnt/sys/
```

`sudo chroot /mnt` will place us as root in a shell in the new Ubuntu installation. Editing of files can be done from another editor outside the chroot or you can install an editor like nano and edit from within.

If you use a non US-keyboard run this to change the keyboard layout:
`dpkg-reconfigure keyboard-configuration`.
To set the correct timezone use:
`dpkg-reconfigure tzdata`.
If you see complaints about locale run:
`dpkg-reconfigure locales`
and check any locales that are relevant to you.

At this point we also create a non root user for the system: `adduser <username>` followed by `usermod -aG sudo <username>` which adds the new user to the *sudo* group. I also like to edit the configuration for the SSH server (`/etc/ssh/sshd_config`) if I will be using it to disallow login with password. For a portable system such as this changing the port for sshd can prevent conflict with the sshd that normally runs on the system. If using the same port clients will detect that the host keys are different and complain. After updating it checking that the configuration file is valid can be done using `sudo sshd -t`.

To get newer security updates we need to add the line `deb http://security.ubuntu.com/ubuntu bionic-security main universe` to one of the files APT uses to determine its repositories, e.g `/etc/apt/sources.list` or a file in `/etc/apt/sources.list.d/`. The same repositories (main, universe etc.) should be enabled here as in the previous line. We can also add `bionic-updates` to get newer versions of some packages.

`/etc/apt/sources.list` on USB-drive:
```
deb http://archive.ubuntu.com/ubuntu bionic main universe
deb http://security.ubuntu.com/ubuntu bionic-security main universe
deb http://archive.ubuntu.com/ubuntu bionic-updates main universe
#deb-src http://archive.ubuntu.com/ubuntu bionic main universe
```

Run `apt update` to use the new repositories. We proceed to install the kernel and bootloader:

`apt install linux-generic-hwe-18.04 grub-efi-amd64 grub-pc-bin`

We can also install cryptsetup if using encryption and standard^ to get a collection of usually installed Ubuntu packages:

`apt install cryptsetup standard^`

If a menu pops about about installing grub as this point you can safely exit it without installing since installing grub is our next step.

`grub-install --target=i386-pc /dev/<usb-drive>`

`grub-install --target=x86_64-efi --removable --no-nvram  /dev/<usb-drive>`

The usb drive is the drive itself and not a partition on it so for example /dev/sdc rather than /dev/sdc4.

To make sure everything is using the right configuration we can run
`sudo update-initramfs -u; sudo update-grub`

### Network
Simple configuration to leave everything to NetworkManager. This can be put in a file with a .yaml  extension in `/etc/netplan/` [[1]](https://netplan.io/examples)
```
network:
  version: 2
  renderer: NetworkManager
```

### Desktop environment
A simple way to get a useable system (Wi-Fi support and other useful things) quickly is to install something like Mate. I use Mate because I think it runs fast enough from my USB-drive while also providing a lot of good features like a menu with search. it is hardly minimal though so if you are short on space you could try finding a smaller collection of packages that provide a useful desktop.

`apt install ubuntu-mate-desktop`

Alternatives such as kubuntu-desktop and xubuntu-desktop exist. You can use whatever desktop environment you like (or not use one) but I recommend something fairly light-weight both for reasons of storage space and speed when running from a USB-drive.

Before removing the USB I recommend creating a temporary script to easily mount everything again if the USB-drive turns out not to function the first time. Make sure to store it somewhere that survives a reboot and it not on any of the partitions mounted by the script.
```
cryptsetup open /dev/sdc4 mobile-crypt
mount /dev/mapper/mobile-crypt /mnt/
mount /dev/sdc3 /mnt/boot
mount /dev/sdc2 /mnt/boot/efi

mount --bind /dev/ /mnt/dev/
mount --bind /tmp/ /mnt/tmp/
mount --bind /proc/ /mnt/proc/
mount --bind /sys/ /mnt/sys/
```

Now it is time to make the USB removable. Exit the chroot session.
Run `sudo umount -R /mnt` to unmount all the file systems under `/mnt`. Then use `sudo cryptsetup close mobile-crypt` to close the LUKS partition.

# Post chroot setup
These steps are done when booting from the USB-drive. Because the chroot makes it easy to install most packages we installed them before booting into out new system. Now it is time to verify that we can boot and that things work as expected. I also like to install Gparted and ddrescue (package name gddrescue) because they are useful on a portable system if using it to repair or recover data. Other data recovery tools can be installed but ddrescue is good for creating an image which can be worked on safely later.

We can set the hostname at this point using: `hostnamectl set-hostname <hostname>`
If the network is not working you can try `sudo netplan apply` to make sure the network configuration is used.


# Troubleshooting and additional notes
If you are not asked for a password on startup when using encryption, verify that cryptsetup is installed on the USB-drive. Run `sudo update-initramfs -uv` and look for lines with hook and crypt in them to make sure that the tools for unlocking at boot time get included in initramfs. You can also try changing the *splash* in `/etc/default/grub` to *nosplash* (and do an update-grub) which I have found to sometimes fix this issue. This is an example line: `GRUB_CMDLINE_LINUX_DEFAULT=quiet nosplash`

You may need to disable secure boot to boot the USB-drive on a machine. I tried installing the signed version of GRUB but for some reason my PC did not boot the USB with secure boot enabled.

Credit to some additional sources I got some information from when I made this.
Unfortunately I did not plan to make this a blog project from the start so I did not document them 100%.

# Additional links and sources
* https://blog.heckel.xyz/2017/05/28/creating-a-bios-gpt-and-uefi-gpt-grub-bootable-linux-system/
* https://help.ubuntu.com/lts/installation-guide/powerpc/apds04.html
* https://wiki.archlinux.org/index.php/Chroot
