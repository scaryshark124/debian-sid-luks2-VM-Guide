# Build Debian Sid using debootsrap with Argon2id and Secure Boot Guide
I am going to demonstrate how to use debootstrap to build Debian Sid on a VirtualBox VM. While in chroot, I will demonstrate how to compile grub2 with ArchLinux's grub-improved-luks2 that allows me to encrypt the root system with argon2id password hashing algorithm. Lastly, I will demonstrate how to create machine owner key (MOK) to sign grub bootloader and vmlinuz.

### What you need:
Debian live standard ISO (no GUI needed) - [here](https://cdimage.debian.org/cdimage/weekly-live-builds/amd64/iso-hybrid/?C=S;O=D)

You will need to have VirtualBox installed in order to follow this guide. I will not cover how to do this. However, I will drop a link to [download](https://www.virtualbox.org/wiki/Linux_Downloads)

## Creating the Virtual Machine
You will need to first create a Virtualbox VM. In the initial creation menu, Select the ISO you downloaded. I used Debian 64-bit version. As for Hardware, I gave it 2GB RAM with 4 CPU cores. I used VHD (Virtual Hard Disk) as the Hard Disk File Type and Variant. After you create the VM there are a few settings that need to be changed. Under Motherboard make sure to Enable EFI (special OSes only). Do not enable Secure Boot yet, as we will mount an unsigned ISO when UEFI fails to find a bootloader. Finally, under Network, change from NAT to Bridged Adapter, this is so we can SSH into the VM from the host system. 

After you start the VM for the first time the UEFI will give you an error becuase it cannot find a bootloader. Next, VirtualBox should give you a pop-up window that allows you to select the debian standard ISO that was downloaded earlier. After it reboots, select Live System (amd64). 

These commands will be done from the VM terminal:

```
sudo apt install openssh-server net-tools
```
Next, we will give user a password. 

```
sudo passwd user
```
We will need the IP address of the VM so that we can ssh from the host. You will want the inet.

```
sudo ifconfig
```
## From the Host System

The remainder of the guide we will be working from the host system. So, open up a terminal on the host:

```
ssh user@192.168.35.73

user@debian:~$ sudo su -

root@debian:~# lsblk
NAME  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0  1.2G  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs
                               /run/live/rootfs/filesystem.squashfs
sda     8:0    0 30.1G  0 disk 
sr0    11:0    1  1.7G  0 rom  /usr/lib/live/mount/medium
                               /run/live/medium
```
I will create a directory and variable for the mount point. 

```
root@debian:~# mkdir -vp /mnt/sid

root@debian:~# echo 'CB="/mnt/sid"' >> ~/.bashrc

root@debian:~# source ~/.bashrc
```

I will install what we will need during the build. I use arch-install-scripts because it is quicker to get in and out of chroot, and genfstab cuts out the hassle of manually configuring /etc/fstab.
```
root@debian:~# apt install arch-install-scripts debootstrap gdisk cryptsetup dosfstools -y
```


I will prepare the virtual hard drive by creating the EFI and root partitions.

```
root@debian:~# gdisk /dev/sda
GPT fdisk (gdisk) version 1.0.10

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-63173854, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-63173854, default = 63172607) or {+-}size{KMGTP}: +200M
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): ef00
Changed type of partition to 'EFI system partition'

Command (? for help): n
Partition number (2-128, default 2): 
First sector (34-63173854, default = 411648) or {+-}size{KMGTP}: 
Last sector (411648-63173854, default = 63172607) or {+-}size{KMGTP}: 
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sda: 63173888 sectors, 30.1 GiB
Model: VBOX HARDDISK   
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): D17DD206-A5E4-4494-9BB8-B1C96A74D5B6
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 63173854
Partitions will be aligned on 2048-sector boundaries
Total free space is 3261 sectors (1.6 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          411647   200.0 MiB   EF00  EFI system partition
   2          411648        63172607   29.9 GiB    8300  Linux filesystem

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
The operation has completed successfully.
```

## Creating the Luks2 encrypted partition

I will start encrypting partitions. I will be using luks2 with argon2id. 

```
root@debian:~# cryptsetup luksFormat --pbkdf=argon2id --use-urandom -s 512 -h sha512 -i 5000 /dev/sda2

WARNING!
========
This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sda2: 
Verify passphrase: 
```
I will open the newly created encrypted partition:
```
root@debian:~# cryptsetup open /dev/sda2 sid-cryptlvm
Enter passphrase for /dev/sda2: 
```
 
## Creating Physical volume, Volume group, Logical Volume

I will create the physical volume, volume group, and logical volumes for the swap and root partition:
```
root@debian:~# pvcreate /dev/mapper/sid-cryptlvm
  Physical volume "/dev/mapper/sid-cryptlvm" successfully created.

root@debian:~# vgcreate vg0 /dev/mapper/sid-cryptlvm
  Volume group "vg0" successfully created
 		
root@debian:~# lvcreate -L 4G vg0 -n swap
  Logical volume "swap" created.
		
root@debian:~# lvcreate -l +100%FREE vg0 -n root
  Logical volume "root" created.

root@debian:~# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
loop0              7:0    0  1.2G  1 loop  /usr/lib/live/mount/rootfs/filesystem.squashfs
                                           /run/live/rootfs/filesystem.squashfs
sda                8:0    0 30.1G  0 disk  
├─sda1             8:1    0  200M  0 part  
└─sda2             8:2    0 29.9G  0 part  
  └─sid-cryptlvm 253:0    0 29.9G  0 crypt 
    ├─vg0-swap   253:1    0    4G  0 lvm   
    └─vg0-root   253:2    0 25.9G  0 lvm   
sr0               11:0    1  1.7G  0 rom   /usr/lib/live/mount/medium
                                           /run/live/medium
```

## Creating the File systems

I will create the filesystems for the EFI, swap, and root partitions. 
```
root@debian:~# mkfs.fat -F 32 /dev/sda1
mkfs.fat 4.2 (2021-01-31)
 
root@debian:~# mkswap -L swap /dev/vg0/swap
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
LABEL=swap, UUID=76d6e79d-eac8-4f16-9ab4-5dd74e798f6a

root@debian:~# mkfs.ext4 -L 'root' /dev/vg0/root
mke2fs 1.47.2-rc1 (28-Nov-2024)
Creating filesystem with 6792192 4k blocks and 1700608 inodes
Filesystem UUID: 0b82a16d-e20f-4cd7-9e56-02c55d22861b
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done 
```

## Mounting the root and EFI partitions

I will mount the newly created filesystems to the host in preparation to build. 

As a sanity check:
```
root@debian:~# echo $CB
/mnt/sid
```

now create and mount:
```
root@debian:~# mount -v /dev/vg0/root $CB
mount: /dev/mapper/vg0-root mounted on /mnt/sid.

root@debian:~# mkdir -vp $CB/efi
mkdir: created directory '/mnt/sid/efi'

root@debian:~# mount -v /dev/sda1 $CB/efi
mount: /dev/sda1 mounted on /mnt/sid/efi.
```
I will turn on the swap for the build.

```
root@debian:~# swapon /dev/mapper/vg0-swap
```
If everything went alright, you should a similar output as below running 'lsblk':
```
root@debian:~# lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
loop0              7:0    0  1.2G  1 loop  /usr/lib/live/mount/rootfs/filesystem.squashfs
                                           /run/live/rootfs/filesystem.squashfs
sda                8:0    0 30.1G  0 disk  
├─sda1             8:1    0  200M  0 part  /mnt/sid/efi
└─sda2             8:2    0 29.9G  0 part  
  └─sid-cryptlvm 253:0    0 29.9G  0 crypt 
    ├─vg0-swap   253:1    0    4G  0 lvm   [SWAP]
    └─vg0-root   253:2    0 25.9G  0 lvm   /mnt/sid
sr0               11:0    1  1.7G  0 rom   /usr/lib/live/mount/medium
                                           /run/live/medium
```
If everything has went well so far you should be able to run this:

```
root@debian:~# debootstrap --components=main,contrib,non-free,non-free-firmware sid $CB http://ftp.us.debian.org/debian/
```

I will create a few things before using chroot. I will use a tool from arch-install-scripts to generate an fstab:
```
root@debian:~# genfstab -U $CB >> $CB/etc/fstab
```

I will create the hostname and hosts files:
```
root@debian:~# echo 'crunchy-sid' > $CB/etc/hostname

root@debian:~# cat > $CB/etc/hosts << EOF
127.0.0.1	localhost
127.0.1.1	crunchy-sid
::1  		localhost ip6-localhost ip6-loopback
ff02::1  	ip6-allnodes
ff02::2  	ip6-allrouters
EOF
```

I will use arch-chroot from arch-install-scripts. This will automatically mount the psuedo filesystems and the resolv.conf for internet connection. arch-chroot will also automatically unmount when exiting chroot. This streamlines the chroot process.
```
root@debian:~# arch-chroot /$CB /bin/bash --login
```
I will set a new passowrd for root.
```
root@debian:/# passwd
New password: 
Retype new password: 
passwd: password updated successfully
```
I will make it where apt package manager won't install grub as we will be compling grub with argon2 from source.

```
root@debian:/# apt-mark hold grub2
grub2 set on hold.
```
I will install a locales and reconfigure tzdata. for locales, I will use en_US.UTF-8, this will need to be changed based on your location. I reconfigured my tzdata to reflect my own timezone, this may be different as it is based on your location. 
```
root@debian:/# apt update && apt upgrade

root@debian:/# apt install locales -y

root@debian:/# dpkg-reconfigure locales
Locales to be generated: 97
Default locale for the system environment: 2
Generating locales (this might take a while)...
  en_US.UTF-8... done
Generation complete.

root@debian:/# dpkg-reconfigure tzdata
Local time is now:      Sun Jan  5 21:20:36 CST 2025.
Universal Time is now:  Mon Jan  6 03:20:36 UTC 2025.
```  
I will install tools needed for initramfs and cryptsetup so that I can create /etc/crypttab for the encrypted root partition. 
```
root@debian:/# apt install initramfs-tools cryptsetup cryptsetup-initramfs -y
```
I will create the resume file in initramfs-tools conf.d directory (optional)
```
root@debian:/# grep '[[:blank:]]swap' /etc/fstab | grep UU | cut -c '6-41'
76d6e79d-eac8-4f16-9ab4-5dd74e798f6a

root@debian:/# > /etc/initramfs-tools/conf.d/resume printf 'RESUME=%s\n' "$(grep '[[:blank:]]swap' /etc/fstab | grep UU | cut -c '-41')"

root@debian:/# cat /etc/initramfs-tools/conf.d/resume
RESUME=UUID=76d6e79d-eac8-4f16-9ab4-5dd74e798f6a
```
I will install the linux headers and linux image along with the logical volume manager
```
root@debian:/# apt install --fix-missing lvm2 linux-headers-amd64 linux-image-amd64
```

 I will install what I need to compile grub from source:
```
root@debian:/# apt install --fix-missing shim-signed shim-helpers-amd64-signed sudo git curl libarchive-tools help2man python3 rsync texinfo ttf-bitstream-vera build-essential dosfstools efibootmgr uuid-runtime efivar mtools os-prober dmeventd libdevmapper-dev libdevmapper-event1.02.1 libdevmapper1.02.1 libfont-freetype-perl python3-freetype libghc-gi-freetype2-dev libghc-gi-freetype2-prof fuse2fs libconfuse2 gettext xorriso libisoburn-dev autogen gnulib libfreetype-dev pkg-config m4 libtool automake flex fuse3 libfuse3-dev gawk autoconf-archive rdfind fonts-dejavu lzma liblzma5 liblzma-dev liblz1 liblz-dev unifont acl libzfslinux-dev sbsigntool texinfo libfuse2 libisoburn1 libtasn1-6 libtasn1-6-dev libtasn1-bin
```  
  I will create the keys directory:
  ```
root@debian:/# mkdir -vp /etc/keys
mkdir: created directory '/etc/keys'
```
Now I will make sure the luk2 volume gets unlocked during boot. Here is my take on including a key to unlock the root partition during the initramfs stage. So, during the boot process grub runs 'cryptomount -u SOME-UUID', this is the first prompt for password for grub to access the /boot directory. I want to note, that during this time the entire partition is unlocked, not just the /boot directory. So, if someone were able to get by the first password prompt and decyrpt the luks2 partition, then having the key is useless. As they will already have control over the entire luks2 partition, not just the /boot directory. I will take measures to ensure proper handling of the key during the initramfs stage. 
```
root@debian:/# ( umask 0077 && dd if=/dev/urandom bs=1 count=128 of=/etc/keys/root.key conv=excl,fsync )
128+0 records in
128+0 records out
128 bytes copied, 0.0195378 s, 6.6 kB/s
```
To make sure to set permissions so that only the owner can read, write, and execute. I will also set immutable.
```
root@debian:/# chmod -vR 600 /etc/keys
mode of '/etc/keys' changed from 0755 (rwxr-xr-x) to 0600 (rw-------)
mode of '/etc/keys/root.key' retained as 0600 (rw-------)

root@debian:/# chattr +i /etc/keys/root.key
```  
I will add the key:
```
root@debian:/# cryptsetup --cipher aes-xts-plain64:sha512 -s 512 -h sha512 --pbkdf=argon2id luksAddKey /dev/sda2 /etc/keys/root.key
Enter any existing passphrase: 
```

I will now configure /etc/crypttab.
```
root@debian:/# echo "sid-cryptlvm UUID=$(blkid -s UUID -o value /dev/sda2) /etc/keys/root.key luks,discard,key-slot=1" >> /etc/crypttab

root@debian:/# cat /etc/crypttab
# <target name>	<source device>		<key file>	<options>
sid-cryptlvm UUID=10618c03-dd6e-4ce2-bcaf-fed849f79145 /etc/keys/root.key luks,discard,key-slot=1
```
I will add this line so initramfs is able to find the key
```
root@debian:/# echo "KEYFILE_PATTERN=\"/etc/keys/*.key\"" >>/etc/cryptsetup-initramfs/conf-hook
```
I will set UMASK to restrictive value to avoid leaking key material. I will then make sure restrictive permissions are set and the key is available in initramfs.
```
root@debian:/# echo UMASK=0077 >>/etc/initramfs-tools/initramfs.conf
```
For some reason Debian installs grub2... run these commands to change the name of "grub-install" and "grub-mkconfig". This must be done because when we compile grub from source it will not replace them if they already exist.

```
root@crunchy-kaisen:/# mv /usr/sbin/grub-install /usr/sbin/grub-install.original

root@crunchy-kaisen:/# mv /usr/sbin/grub-mkconfig /usr/sbin/grub-mkconfig.original
```
This can be customized depending on how you want to setup your sysetm. 
```
root@debian:/# apt install --fix-missing --install-recommends firmware-linux-free firmware-linux firmware-linux-nonfree task-xfce-desktop base-files
```
check to make sure restrictive permissions are set:
```
root@debian:/# stat -L -c "%A %n" /initrd.img
-rw------- /initrd.img
```
check to make sure the key is included in initramfs:
```
root@debian:/# lsinitramfs /initrd.img | grep "^cryptroot/keyfiles/"
cryptroot/keyfiles/kaisen-cryptlvm.key
```

I will create a user:
```
root@debian:/# useradd -mG cdrom,floppy,sudo,audio,dip,video,plugdev,netdev -s /usr/bin/bash -c 'Crunchy Taco' crunchy
```
I will now create a password for the new user:
```
root@debian:/#  passwd crunchy
New password: 
Retype new password: 
passwd: password updated successfully
```

You have to include gcc in the PATH variable or the compile will fail for grub. The other ones are so grub can find the extra libraries that get cloned in for the build. The CFLAGS is something i got from a script that i need to find again. 
```
root@debian:/#  export PATH="$PATH:/bin/gcc:/sbin/gcc"
export GRUB_CONTRIB=./grub-extras
export GNULIB_SRCDIR=./gnulib
export CFLAGS=${CFLAGS/-fno-plt}
```

A work-around for mawk giving errors during make is to rename mawk in /usr/bin and then point mawk to gawk: 
```
root@debian:/# mv /usr/bin/mawk /usr/bin/mawk_bu

root@debian:/# ln -s /usr/bin/gawk /usr/bin/mawk
```

I will create a sources folder and grub direcotry in /boot for the grub build:
```
root@debian:/# mkdir -vp /boot/grub

root@debian:/# mkdir -vp /sources && cd sources
```


Cloning git repositories needing for Archlinux's argon2 patch. 
```
root@debian:/sources# git clone https://git.savannah.gnu.org/git/grub.git
Cloning into 'grub'...
remote: Counting objects: 101945, done.
remote: Compressing objects: 100% (23535/23535), done.
remote: Total 101945 (delta 76259), reused 101518 (delta 75970)
Receiving objects: 100% (101945/101945), 71.95 MiB | 8.46 MiB/s, done.
Resolving deltas: 100% (76259/76259), done.
    
root@debian:/sources# cd grub

root@debian:/sources/grub# git clone https://git.savannah.nongnu.org/git/grub-extras.git
Cloning into 'grub-extras'...
remote: Counting objects: 949, done.
remote: Compressing objects: 100% (332/332), done.
remote: Total 949 (delta 563), reused 949 (delta 563)
Receiving objects: 100% (949/949), 1.48 MiB | 3.37 MiB/s, done.
Resolving deltas: 100% (563/563), done.

root@debian:/sources/grub#  git clone https://aur.archlinux.org/grub-improved-luks2-git.git
Cloning into 'grub-improved-luks2-git'...
remote: Enumerating objects: 112, done.
remote: Counting objects: 100% (112/112), done.
remote: Compressing objects: 100% (65/65), done.
remote: Total 112 (delta 46), reused 110 (delta 46), pack-reused 0 (from 0)
Receiving objects: 100% (112/112), 94.96 KiB | 330.00 KiB/s, done.
Resolving deltas: 100% (46/46), done.

root@debian:/sources/grub#  git clone https://git.savannah.gnu.org/git/gnulib.git
Cloning into 'gnulib'...
remote: Counting objects: 298733, done.
remote: Compressing objects: 100% (36559/36559), done.
remote: Total 298733 (delta 264658), reused 296054 (delta 262064)
Receiving objects: 100% (298733/298733), 77.15 MiB | 5.73 MiB/s, done.
Resolving deltas: 100% (264658/264658), done.
```

This part is copied from grub-improved-luks2-git/PKGBUILD. It patches grub and compiles and installs it
```
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/add-GRUB_COLOR_variables.patch
patching file util/grub-mkconfig.in
Hunk #1 succeeded at 250 (offset 4 lines).
patching file util/grub.d/00_header.in
```

Patch grub-mkconfig to detect Arch Linux initramfs images.
```
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/detect-archlinux-initramfs.patch
patching file util/grub.d/10_linux.in
Hunk #1 succeeded at 95 (offset 2 lines).
Hunk #2 succeeded at 212 (offset 12 lines).
Hunk #3 succeeded at 301 with fuzz 1 (offset 14 lines).
```

Argon2:
```
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_1.patch
patching file grub-core/kern/dl.c
Hunk #1 succeeded at 470 (offset 3 lines).
patching file util/grub-module-verifierXX.c
Hunk #1 succeeded at 236 with fuzz 1 (offset 79 lines).

root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_2.patch
patching file include/grub/types.h
Hunk #1 succeeded at 156 (offset 3 lines).
Hunk #2 succeeded at 178 (offset 3 lines).

root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_3.patch
patching file docs/grub-dev.texi
Hunk #1 succeeded at 503 (offset 13 lines).
patching file grub-core/Makefile.core.def
Hunk #1 succeeded at 1219 (offset 45 lines).
patching file grub-core/lib/argon2/LICENSE
patching file grub-core/lib/argon2/argon2.c
patching file grub-core/lib/argon2/argon2.h
patching file grub-core/lib/argon2/blake2/blake2-impl.h
patching file grub-core/lib/argon2/blake2/blake2.h
patching file grub-core/lib/argon2/blake2/blake2b.c
patching file grub-core/lib/argon2/blake2/blamka-round-ref.h
patching file grub-core/lib/argon2/core.c
patching file grub-core/lib/argon2/core.h
patching file grub-core/lib/argon2/ref.c

root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_4.patch
patching file grub-core/disk/luks2.c
Hunk #1 succeeded at 38 (offset -2 lines).
Hunk #2 succeeded at 91 (offset -2 lines).
Hunk #3 succeeded at 161 (offset -2 lines).
Hunk #4 succeeded at 461 (offset 14 lines).

root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_5.patch
patching file Makefile.util.def
patching file grub-core/Makefile.core.def
Hunk #1 succeeded at 1242 (offset 45 lines).
patching file grub-core/disk/luks2.c
Hunk #2 succeeded at 463 (offset 14 lines).
```

Make grub-install work with luks2:
```
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/grub-install_luks2.patch
patching file util/grub-install.c
Hunk #1 succeeded at 448 (offset 2 lines).
```

Fix DejaVuSans.ttf location so that grub-mkfont can create *.pf2 files for starfield theme:
```
root@debian:/sources/grub#  sed 's|/usr/share/fonts/dejavu|/usr/share/fonts/dejavu /usr/share/fonts/truetype/dejavu|g' -i "configure.ac"
```

Modify grub-mkconfig behaviour to silence warnings FS#36275
```    
root@debian:/sources/grub#  sed 's| ro | rw |g' -i "util/grub.d/10_linux.in"
```
Modify grub-mkconfig behaviour so automatically generated entries read 'Arch Linux' FS#33393
```
root@debian:/sources/grub#  sed 's|GNU/Linux|Linux|' -i "util/grub.d/10_linux.in"
```
Remove lua module from grub-extras as it is incompatible with changes to grub_file_open. http://git.savannah.gnu.org/cgit/grub.git/commit/?id=ca0a4f689a02c2c5a5e385f874aaaa38e151564e
```
root@debian:/sources/grub#  rm -rf ./grub-extras/lua
```
I will bootstrap grub2, then build it:
```
root@debian:/sources/grub#  ./bootstrap

root@debian:/sources/grub#  mkdir ./build_x86_64-efi && cd ./build_x86_64-efi

root@debian:/sources/grub/build_x86_64-efi#  ../configure --with-platform=efi --target=x86_64 --prefix="/usr" --sbindir="/usr/bin" --sysconfdir="/etc" --enable-boot-time --enable-cache-stats --enable-device-mapper --enable-grub-mkfont --enable-grub-mount --enable-mm-debug --disable-silent-rules --disable-werror  CPPFLAGS="$CPPFLAGS -O2" --enable-stack-protector --enable-liblzma
```
You should get something like this:
```
*******************************************************
GRUB2 will be compiled with following components:
Platform: x86_64-efi
With devmapper support: Yes
With memory debugging: Yes
With disk cache statistics: Yes
With boot time statistics: Yes
efiemu runtime: No (not available on efi)
grub-mkfont: Yes
grub-mount: Yes
grub-protect: Yes
starfield theme: Yes
With DejaVuSans font from /usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
With libzfs support: Yes
Build-time grub-mkfont: Yes
With unifont from /usr/share/fonts/X11/misc/unifont.pcf.gz
With liblzma from -llzma (support for XZ-compressed mips images)
With stack smashing protector: Yes
*******************************************************
```

If everything looks alright, then we install grub2:
```
root@debian:/sources/grub/build_x86_64-efi#  make DESTDIR=/ bashcompletiondir=/usr/share/bash-completion/completions install
        
root@debian:/sources/grub/build_x86_64-efi#  install -D -m0644 ../grub-improved-luks2-git/grub.default /etc/default/grub

root@debian:/sources/grub/build_x86_64-efi#  cd ../../..
```

Change the menu title to reflect Kaisen. 
```
root@debian:/# sed -i 's|GRUB_DISTRIBUTOR="Arch"|GRUB_DISTRIBUTOR="sid"|g' /etc/default/grub
```

Enable cryptodisk to give grub the ability to unlock the encrypted partion at boot time to access initramfs in the /boot directory
```
root@debian:/# sed -i 's|#GRUB_ENABLE_CRYPTODISK=y|GRUB_ENABLE_CRYPTODISK=y|g' /etc/default/grub
```
I will make a new grub config file:
```
root@debian:/# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.12.8-amd64
Found initrd image: /boot/initrd.img-6.12.8-amd64
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

I will create an sbat.csv (this is optional if you plan to use secure boot, make sure to remove --sbat when using grub-install in the next step if you do not want to use secure boot):
```
root@debian:/# cat > /usr/share/grub/sbat.csv << EOF
sbat,1,SBAT Version,sbat,1,https://github.com/rhboot/shim/blob/main/SBAT.md
grub,3,Free Software Foundation,grub,2:2.12-2,https//www.gnu.org/software/grub/
grub.arch,1,Arch Linux,grub,2:2.12-2,https://archlinux.org/packages/core/x86_64/grub/
EOF
```

It is time to install the grub efi:
```
root@debian:/# grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/boot --modules="bli argon2 all_video boot btrfs cat chain configfile echo efifwsetup efinet ext2 fat font gettext gfxmenu gfxterm gfxterm_background gzio halt help hfsplus iso9660 jpeg keystatus loadenv loopback linux ls lsefi lsefimmap lsefisystab lssal memdisk minicmd normal ntfs part_apple part_msdos part_gpt password_pbkdf2 png probe reboot regexp search search_fs_uuid search_fs_file search_label serial sleep smbios squash4 test tpm true video xfs zfs zfscrypt zfsinfo cpuid play cryptodisk gcry_arcfour gcry_blowfish gcry_camellia gcry_cast5 gcry_crc gcry_des gcry_dsa gcry_idea gcry_md4 gcry_md5 gcry_rfc2268 gcry_rijndael gcry_rmd160 gcry_rsa gcry_seed gcry_serpent gcry_sha1 gcry_sha256 gcry_sha512 gcry_tiger gcry_twofish gcry_whirlpool luks luks2 lvm mdraid09 mdraid1x raid5rec raid6rec" --sbat /usr/share/grub/sbat.csv /dev/sda
```

I will move over the signed shim and mok manager, as well as the grubx64.efi that we just created with grub-install. 
```
root@debian:/# mkdir -vp /efi/EFI/BOOT
mkdir: created directory '/efi/EFI/BOOT'

root@debian:/# cp /efi/EFI/sid/grubx64.efi /efi/EFI/BOOT/

root@debian:/# cp /usr/lib/shim/shimx64.efi.signed /efi/EFI/BOOT/bootx64.efi

root@debian:/# cp /usr/lib/shim/mmx64.efi.signed /efi/EFI/BOOT/mmx64.efi
```
## Secure Boot (optional)

If you do not care for secure boot, then you are done. Just skip to the end to umount, and close the luks2 volume.

I will create the machine owner key along with the der cert needed when registering it with MOK manager on the first boot. I will also set restrictive permissions:
```
root@debian:/# mkdir -vp /etc/keys/MOK
mkdir: created directory '/etc/keys/MOK'

root@debian:/# (umask 0077 && openssl req -newkey rsa:2048 -nodes -keyout /etc/keys/MOK/MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=Crunchy-sid567 Crunchy-Taco567/" -out /etc/keys/MOK/MOK.crt)



root@debian:/# (umask 0077 && openssl x509 -outform DER -in /etc/keys/MOK/MOK.crt -out /etc/keys/MOK/crunchy-sid-MOK.cer)

root@debian:/# mkdir -vp /efi/EFI/certs
mkdir: created directory '/efi/EFI/certs'


root@debian:/# cp /etc/keys/MOK/crunchy-sid-MOK.cer /efi/EFI/certs/


root@debian:/# chmod -vR 600 /etc/keys
mode of '/etc/keys' changed from 0755 (rwxr-xr-x) to 0600 (rw-------)
mode of '/etc/keys/root.key' retained as 0600 (rw-------)

root@debian:/# chattr +i /etc/keys/MOK/MOK.key

root@debian:/# chattr +i /etc/keys/MOK/crunchy-Kais-MOK.cer

root@debian:/# chattr +i /etc/keys/MOK/MOK.crt
```
Now I will check the signatures and sign vmlinuz and grubx64. 
```
sbverify --list /boot/$(ls /boot | grep vmlinuz)
signature 1
image signature issuers:
 - /CN=Debian Secure Boot CA
image signature certificates:
 - subject: /CN=Debian Secure Boot Signer 2022 - linux
   issuer:  /CN=Debian Secure Boot CA

root@debian:/# sbsign --key /etc/keys/MOK/MOK.key --cert /etc/keys/MOK/MOK.crt --output /boot/$(ls /boot | grep vmlinuz) /boot/$(ls /boot | grep vmlinuz)
Image was already signed; adding additional signature

root@debian:/# sbverify --list /boot/$(ls /boot | grep vmlinuz)
signature 1
image signature issuers:
 - /CN=Debian Secure Boot CA
image signature certificates:
 - subject: /CN=Debian Secure Boot Signer 2022 - linux
   issuer:  /CN=Debian Secure Boot CA
signature 2
image signature issuers:
 - /CN=Crunchy-sid567 Crunchy-Taco567
image signature certificates:
 - subject: /CN=Crunchy-sid567 Crunchy-Taco567
   issuer:  /CN=Crunchy-sid567 Crunchy-Taco567


sbverify --list /efi/EFI/BOOT/grubx64.efi
No signature table present

root@debian:/# sbsign --key /etc/keys/MOK/MOK.key --cert /etc/keys/MOK/MOK.crt --output /efi/EFI/BOOT/grubx64.efi /efi/EFI/BOOT/grubx64.efi
Signing Unsigned original image

root@debian:/# sbverify --list /efi/EFI/BOOT/grubx64.efi
signature 1
image signature issuers:
 - /CN=Crunchy-sid567 Crunchy-Taco567
image signature certificates:
 - subject: /CN=Crunchy-sid567 Crunchy-Taco567
   issuer:  /CN=Crunchy-sid567 Crunchy-Taco567
```
# Let us finish up...

I will umount the virtual filesystems and file systems. I will turn swap back on and shut it off for the build OS. I will then close the luks2 volumes. I will then restart into UEFI and turn on secure boot to test out everything went alright. 
```
root@debian:/# logout

root@debian:~# swapoff /dev/mapper/vg0-swap

root@debian:~# umount -vr $CB/efi

root@debian:~# umount -vr $CB
```	
now you need to close the Luks volume:
```
root@debian:~# cryptsetup close /dev/vg0/swap

root@debian:~# cryptsetup close /dev/vg0/root

root@debian:~# cryptsetup close sid-cryptlvm
```

Now you just need to reboot and register you key with the MOK manager. the shim will automatically start the MOK manager. Or if you didn't create a machine owner key, then it should just boot up without the mok manager.
