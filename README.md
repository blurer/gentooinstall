Using Hetzner ax41-nvme

Specs:
```
AMD Ryzen 5 3600 Hexa-Core
RAM: 64 GB DDR4
Disk: 2 x 512 GB NVMe SSD  (software-RAID 1)
Connection: 1 GBit/s port
Guaranteed bandwidth: 1 GBit/s
Cost: € 40.46
```

The most important part, the neofetch once booted:
```
$ neofetch
         -/oyddmdhs+:.                bl@bl-dev
     -odNMMMMMMMMNNmhy+-`             ---------
   -yNMMMMMMMMMMMNNNmmdhy+-           OS: Gentoo/Linux x86_64
 `omMMMMMMMMMMMMNmdmmmmddhhy/`        Kernel: 5.15.23-gentoo-x86_64
 omMMMMMMMMMMMNhhyyyohmdddhhhdo`      Uptime: 2 days, 4 hours, 40 mins
.ydMMMMMMMMMMdhs++so/smdddhhhhdm+`    Packages: 374 (emerge)
 oyhdmNMMMMMMMNdyooydmddddhhhhyhNd.   Shell: zsh 5.8
  :oyhhdNNMMMMMMMNNNmmdddhhhhhyymMh   Terminal: /dev/pts/0
    .:+sydNMMMMMNNNmmmdddhhhhhhmMmy   CPU: AMD Ryzen 5 3600 (12) @ 3.600GHz
       /mMMMMMMNNNmmmdddhhhhhmMNhs:   GPU: NVIDIA GeForce GT 710
    `oNMMMMMMMNNNmmmddddhhdmMNhs+`    Memory: 974MiB / 64241MiB
  `sNMMMMMMMMNNNmmmdddddmNMmhs/.
 /NMMMMMMMMNNNNmmmdddmNMNdso:`
+MMMMMMMNNNNNmmmmdmNMNdso/-
yMMNNNNNNNmmmmmNNMmhs+/-`
/hMMNNNNNNNNMNdhs++/-`
`/ohdmmddhys+++/:.`
  `-//////:--.

```
## Getting started
- Boot to rescue
- SSH to the host. 
- Suggest to use screen, incase you get disconnected while you are chrooted. 
- Depending on the prior configuration, you may need to remove software raid. 
- This is not for the systemd install

## Create partitions

### nvme0
```
parted -a optimal /dev/nvme0n1
mklabel gpt
-> yes
unit MiB
mkpart primary 2 100
set 1 bios_grub on
mkpart primary 512 1024
mkpart primary linux-swap(v1) 1024 17408
mkpart primary 17408 -1
set 2 raid on
set 4 raid on
p
```
### nvme 1
```
parted -a optimal /dev/nvme1n1
mklabel gpt
-> yes
unit MiB
mkpart primary 2 100
set 1 bios_grub on
mkpart primary 512 1024
mkpart primary linux-swap(v1) 1024 17408
mkpart primary 17408 -1
set 2 raid on
set 4 raid on
p
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


```

## Get the tar and chroot
```
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20220206T170533Z/stage3-amd64-openrc-20220206T170533Z.tar.xz
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
echo 'MAKEOPTS="-j13"' >> /etc/portage/make.conf
sed -i 's/^COMMON_FLAGS.*/COMMON_FLAGS="-march=native -O2 -pipe"/g' /etc/portage/make.conf
emerge --sync
emerge --ask --verbose --update --deep --newuse @world
emerge vim
emerge htop
emerge mdadm 
emerge sudo
```
### Edit fstab:
```
/dev/md0                /boot           ext2            noauto,noatime  1 2
/dev/md1                /               ext4            noatime         0 1
/dev/nvme0n1p3               none            swap            sw,pri=1        0 0
/dev/nvme1n1p3               none            swap            sw,pri=1        0 0
```

### Keep it goin
Set the locale, ensure mdadm has the new fstab info, set timezone

```
echo 'LANG="en_US.utf8"' >> /etc/locale.conf
locale-gen
mdadm --detail --scan >> /etc/mdadm.conf
echo "Asia/Tokyo" > /etc/timezone
emerge --config sys-libs/timezone-data
nano -w /etc/locale.gen
en_US.UTF-8 UTF-8
eselect locale list
eselect locale set 4
env-update && source /etc/profile
ACCEPT_LICENSE="*"
```
## Kernel
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
grub-install /dev/nvme0n1
grub-mkconfig -o /boot/grub/grub.cfg
```
Included the snipped of successful run. Keeping it as important for successful booting. 
```
rescue /etc/init.d # grub-install /dev/nvme1n1
Installing for i386-pc platform.
Installation finished. No error reported.
rescue /etc/init.d # grub-install /dev/nvme0n1
Installing for i386-pc platform.
Installation finished. No error reported.
rescue /etc/init.d # grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.15.23-gentoo-x86_64
Found initrd image: /boot/initramfs-5.15.23-gentoo-x86_64.img
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
done
```
## Last stuff
Core utils for me
```
rc-service add sshd default
useradd bl
usermod -aG wheel bl
mkhomedir_helper bl
passwd bl -> supersecretpw
visudo -> uncomment wheel to have sudo perms
```

Thats it. 
