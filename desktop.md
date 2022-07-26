Gentoo Install on Desktop
* Ryzen 3, 3200g
* 32gb DDR4 sodimm/3200mhz
* 1tb Samsung NVME
* 2x 1tb Samsung SATA SSD
* ASROCK A300


```
## Getting started
- Boot to Arch Install
- Read the Gentoo Wiki -> https://wiki.gentoo.org/wiki/Main_Page
- Read the Arch Wiki for good measure -> https://wiki.archlinux.org/

## Create partitions

fdisk /dev/nvme0n1
> delete everything

n -> +256G -> T: 1 (efi)
n -> +8G -> T: swap
n -> remainder  -> T: linux fx

# Craete file systems



```
### confirm
```
root@rescue ~ # lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0         7:0    0     3G  1 loop
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:6    0    98M  0 part
├─nvme0n1p2 259:7    0   512M  0 part
├─nvme0n1p3 259:8    0    16G  0 part
└─nvme0n1p4 259:9    0 459.9G  0 part
nvme1n1     259:4    0 476.9G  0 disk
├─nvme1n1p1 259:10   0    98M  0 part
├─nvme1n1p2 259:11   0   512M  0 part
├─nvme1n1p3 259:12   0    16G  0 part
└─nvme1n1p4 259:13   0 459.9G  0 part
```
## Create raid
```
mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme0n1p1 /dev/nvme1n1p1
mdadm --create /dev/md1 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme1n1p2 /dev/nvme0n1p2
mdadm --create /dev/md2 --level=1 --raid-devices=2 /dev/nvme1n1p4 /dev/nvme0n1p4
```

## Make filesystems
```
mkfs.ext2 /dev/md0
mkfs.ext4 /dev/md1
mkswap /dev/nvme0n1p3 && mkswap /dev/nvme1n1p3
swapon -p 1 /dev/nvme0n1p3 && swapon -p 1 /dev/nvme1n1p3
mkfs.ext4 /dev/md2
```

### Confirm
```
root@rescue ~ # lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
loop0         7:0    0     3G  1 loop
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:6    0    98M  0 part
│ └─md0       9:0    0  97.9M  0 raid1
├─nvme0n1p2 259:7    0   512M  0 part
│ └─md1       9:1    0 511.9M  0 raid1
├─nvme0n1p3 259:8    0    16G  0 part  [SWAP]
└─nvme0n1p4 259:9    0 459.9G  0 part
  └─md2       9:2    0 459.8G  0 raid1
nvme1n1     259:4    0 476.9G  0 disk
├─nvme1n1p1 259:10   0    98M  0 part
│ └─md0       9:0    0  97.9M  0 raid1
├─nvme1n1p2 259:11   0   512M  0 part
│ └─md1       9:1    0 511.9M  0 raid1
├─nvme1n1p3 259:12   0    16G  0 part  [SWAP]
└─nvme1n1p4 259:13   0 459.9G  0 part
  └─md2       9:2    0 459.8G  0 raid1
  ```

  ## Mount Disks
```
mkdir /mnt/gentoo
mount /dev/md3 /mnt/gentoo/
mkdir /mnt/gentoo/boot
mount /dev/md1 /mnt/gentoo/boot/
cd /mnt/gentoo/
pacman -Sy ; pacman -S links


```

## Get the tar and chroot
```
links https://www.gentoo.org/downloads
-> stage3 -> openrc -> save
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner

cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

```
## Install install
```
mkdir /etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf
emerge-webrsync
echo 'MAKEOPTS="-j5"' >> /etc/portage/make.conf
echo 'ACCEPT_LICENSE="*"' >> /etc/portage/make.conf
sed -i 's/^COMMON_FLAGS.*/COMMON_FLAGS="-march=native -O2 -pipe"/g' /etc/portage/make.conf
emerge --sync
emerge --ask --verbose --update --deep --newuse @world
emerge vim htop sudo
```
### Edit fstab:
```
/dev/nvme0n1p1          /boot           vfat            noauto,noatime  1 2
/dev/nvme0n1p3          /               ext4            noatime         0 1
/dev/nvme0n1p2          none            swap            sw,pri=1        0 0
```

### Keep it goin
Set the locale, ensure mdadm has the new fstab info, set timezone

```
echo 'LANG="en_US.utf8"' >> /etc/locale.conf
locale-gen
echo "Asia/Tokyo" > /etc/timezone
emerge --config sys-libs/timezone-data
nano -w /etc/locale.gen
en_US.UTF-8 UTF-8
eselect locale list
eselect locale set 4 (OR whatever has en_US-UTF8)
env-update && source /etc/profile
```
## Kernel
Pick bin or compile

### Kernel - Bin
```
emerge gentoo-kernel-bin
```

### Kernel - Genkernel
Emerge the kernel, grub, and some utils
```
emerge sys-kernel/linux-firmware sys-kernel/genkernel gentoo-sources nload iftop grub dev-vcs/git
sed -i 's/.*MICROCODE=.*/MICROCODE="yes"/' /etc/genkernel.conf
sed -i 's/.*SSH=.*/SSH="yes"/' /etc/genkernel.conf
sed -i 's/.*BUSYBOX=.*/BUSYBOX="yes"/' /etc/genkernel.conf
sed -i 's/.*MDADM=.*/MDADM="yes"/' /etc/genkernel.conf
sed -i 's/.*E2FSPROGS=.*/E2FSPROGS="yes"/' /etc/genkernel.conf
```
Not sure if this a bug when having ssh enabled in the kernel, more to follow, but I had to add a ssh key to ``/etc/dropbear/authorized_keys`` otherwise I got an error saying sshkeys are missing. After that, generate the kernel. Grab a coffee or two (~20 min on this system).
```
genkernel all
```

## Install bootloader and run it
```
grub-install /dev/nvme1n1
grub-mkconfig -o /boot/grub/grub.cfg

## Final system setup

```
passwd root -> supersecretpw
```

Core utils for me
```
rc-service add sshd default
useradd bl
usermod -aG wheel bl
mkhomedir_helper bl
passwd bl -> supersecretpw
visudo -> uncomment wheel to have sudo perms
```

Boot it up and login as user or root.

sudo emerge x11-terms/st app-editors/vim app-misc/ranger net-misc/ntp x11-misc/dmenu x11-apps/xsetroot x11-vm/dwm x11-base/xorg-server sys-apps/bat sys-process/htop www-client/firefox-bin
