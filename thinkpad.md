# Gentoo Install on Desktop

System
```
* Thinkpad t480s 
* 24gb ddr4
* 1tb SK Hynix NVME
* Intel Wi-Fi AX211
```


## Getting started
- Boot to Arch Install
- Read the Gentoo Wiki -> https://wiki.gentoo.org/wiki/Main_Page
- Read the Arch Wiki for good measure -> https://wiki.archlinux.org/
- ``pacman -S screen wget curl links openssh-server``
- ``iwctl`` -> join your wifi
- ``systemctl start sshd``
- ``passwd`` -> set root pw
- ``ip addr`` -> get ip
- ssh to the host

# Disks

## Create partitions
```
fdisk /dev/nvme0n1
> delete everything
n -> +256M -> T: 1 (efi)
n -> +905.4G -> T: linux fs
n -> +24.6G -> T: swap
```
## Create file systems

```
mkfs.vfat -F 32 /dev/nvme0n1p1
mkswap /dev/nvme1n1p3
swapon -p 1 /dev/nvme0n1p3
mkfs.ext4 /dev/nvme0n1p2
```

## Confirm
```
nvme0n1     259:2    0 931.5G  0 disk
├─nvme0n1p1 259:3    0   256M  0 part /boot
├─nvme0n1p3 259:4    0     8G  0 part [SWAP]
└─nvme0n1p2 259:5    0 923.3G  0 part /
```

## Mount Disks
```
mkdir /mnt/gentoo
mount /dev/nvme0n1p2 /mnt/gentoo/
mkdir /mnt/gentoo/boot
mount /dev/nvme0n1p1 /mnt/gentoo/boot/
cd /mnt/gentoo/
pacman -Sy ; pacman -S links

```
# System
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
## Install
```
mkdir /etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf
emerge-webrsync
echo 'MAKEOPTS="-j7"' >> /etc/portage/make.conf
echo 'ACCEPT_LICENSE="*"' >> /etc/portage/make.conf
sed -i 's/^COMMON_FLAGS.*/COMMON_FLAGS="-march=native -O2 -pipe"/g' /etc/portage/make.conf
emerge --sync
emerge --ask --verbose --update --deep --newuse @world
emerge vim htop sudo
```

## Profile
Set desktop profile

```
eselect profile list
  [2]   default/linux/amd64/17.1/desktop
eselect profile set 2
```

## Edit fstab:
```
/dev/nvme0n1p1          /boot           vfat            noauto,noatime  1 2
/dev/nvme0n1p2          /               ext4            noatime         0 1
/dev/nvme0n1p3          none            swap            sw,pri=1        0 0
```

## Keep it goin
Set the locale, ensure mdadm has the new fstab info, set timezone

```
echo 'LANG="en_US.utf8"' >> /etc/locale.conf
locale-gen
echo "America/New_York" > /etc/timezone
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
emerge gentoo-kernel-bin wpa_supplicant
eselect kernel list
{select the kernel you want - should be 1}
eselect kernel set 1
```

### Confirm Kernel is present in /usr/src/linux
*System wont have a bootable Kernel if this is not accurate*
```
root #ls -l /usr/src/linux
lrwxrwxrwx 1 root root 25 Oct 10 16:08 /usr/src/linux -> linux-5.15.72-gentoo-dist
```

## Install bootloader and make grub config
*If you get errors here, check syntax and/or make sure /boot is mounted to /dev/nvme0n1p1*
```
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
emerge sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
```
### IMPORTANT
Make sure you see the following example output for grub-install and grub-mkconfig. These two not being completed means you will not have a bootable system.
```
Generating grub.cfg ...
Found linux image: /boot/vmlinuz-5.15.52-gentoo
Found initrd image: /boot/initramfs-genkernel-amd64-5.15.52-gentoo
done
```

## System Tools
*These are openrc versions of logging, cron, search, etc*
```
emerge net-misc/dhcpcd iwlwifi net-misc/netifrc app-admin/sysklogd sys-process/cronie sys-apps/mlocate sys-fs/e2fsprogs 
rc-update add sysklogd default
rc-update add cronie default
```
## Network Adapters
```
cd /etc/init.d/
ln -s net.lo net.eth0
ln -s net.lo net.wlp61s0
rc-update add dhcpcd default
rc-update net.eth0 default
rc-update net.wlp61s0 default
```
## Hosts/Hostname
* Set hostname
* Edit /etc/hosts

## User stuff
```
passwd root -> supersecretpw
rc-service add sshd default
useradd bl
usermod -aG wheel bl
mkhomedir_helper bl
passwd bl -> supersecretpw
visudo -> uncomment wheel to have sudo perms
```
### Reboot

## First boot
- Login as user
- Configur wifi

### Wifi connection
*First time requires the manual wpa_supplicant -B .. and dhcpcd, upon reboot itll work if setup ``rc-update net.wlp61s0 default`` earlier.*
```
wpa_passphrase {SSID} {PASSWORD} > /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i wlp61s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
dhcpcd wlp61s0
```

### Install some stuff
```
app-editors/vim \
net-misc/ntp \
x11-misc/dmenu \
x11-apps/xsetroot \
x11-wm/dwm \
x11-base/xorg-server \
sys-process/htop \
gui-libs/display-manager-init \
apps-misc/screen \
lightdm \
alsa-plugins \
ranger \
feh \
firefox-bin \
pavucontrol \
pipewire
```
*Note the emerge may complain about firefox-bin - read the message*

### x11 stuff
```
exec dbus-launch --sh-syntax --exit-with-session dwm
touch $HOME/.Xauthority
rc-update add elogind boot
rc-update add dbus default
rc-update add display-manager default
```
### Edit display-manager 
```
vim /etc/conf.d/display-manager
DISPLAYMANAGER="lightdm"
```

### Install python3-pip
```
wget https://bootstrap.pypa.io/get-pip.py
python get-pip.py
```

### Add user to video
```
sudo usermod -aG video bl
sudo usermod -aG audio bl
```

## Other stuff
```
emerge discord-bin
```
