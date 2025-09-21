# dot files and support scripts for a niri void linux installation

https://voidlinux.org/

I stumbled on to the niri compositor (https://github.com/YaLTeR/niri) while looking through the man pages for waybar to style it for my Chimera linux installation.  I was like, "well, what is niri-window and niri-workspaces (?!?)"  The thing about dual/triple/* booting your system is that sometimes you end up in the wrong installation booted.  And the journey began.

For the most part, these scripts and configs were torn and modified from my other repos.  My installation of this awesome distro starts from the rootfs musl archive, which is processed and unpacked from another running distribution (could be a live usb/cd as well), using ```lvm```, with networking started and booted in UEFI mode.  Here's the long version:

```
# create logical volume
lvcreate -n void_root -L 50G <some volume group>
mkfs.ext4 -L void_root /dev/<some volume group>/void_root

lvcreate -n void_swap -L <a little more than installed system ram> <some volume group>
mkswap -L void_swap /dev/<some volume group>/void_swap
# as super user
cd
mkdir root
mount /dev/<some volume> "$HOME"/root
tar xf void-<arch>-musl-ROOTFS-<date>.tar.xz -C "$HOME"/root

# set up chroot
cd root
mount -t proc proc proc
mount -t sysfs sysfs sys
mount -o bind /dev dev
mount -t devpts devpts dev/pts
mount -t efivarfs efivarfs sys/firmware/efi/efivars
# make sure to clear link in unpacked etc NOT booted system /etc
rm -f etc/resolv.conf
cp /etc/resolv.conf etc/resolv.conf
chroot . /bin/bash

# update
xbps-install -Syu xbps
xbps-install -yu

cd <path of this repo>
# install nano and void-repo-nonfree
for package in $(head -n 2 packages) ; do xbps-install -y "$package" ; done

# update void-repo-nonfree
xbps-install -Sy

# install everything else
for package in $(tail -n+3 packages); do xbps-install -y "$package" ; done

# fix for fbterm
setcap 'cap_sys_tty_config+ep' /usr/bin/fbterm

# remove orphans and unused packages
xbps-remove -RoOy

# ignore and remove sudo and wpa_supplicant
cp ignore-sudo.conf /etc/xbps.d
cp ignore-wpa_supplicant.conf /etc/xbps.d
xbps-remove -Ry sudo wpa_supplicant

# setup a user
useradd -m <user>
usermod -a -G "$(cat user_groups) <user>
passwd user

# edit doas.conf (replace <user> in file with newly created user) and cp to /etc
cp doas.conf /etc

# update root password
passwd

# create fstab and mount in chroot
blkid | grep void_root >> /etc/fstab
blkid | grep EFI >> /etc/fstab
blkid | grep void_swap >> /etc/fstab
# the next step would be to edit fstab. example:
# See fstab(5).
#
# <file system>					<dir>		<type>	<options>			<dump>	<pass>
# root
#UUID=12345678-90ab-cdef-1234-567890abcdef	/		ext4	rw,relatime,errors=remount-ro	0	1
# efi
#UUID=1234-5678					/boot/efi	vfat	umask=0077			0	1
# tmp
#tmpfs						/tmp		tmpfs	defaults,nosuid,nodev		0	0
#swap
#UUID=12345678-90ab-cdef-1234-567890abcdef	swap		swap	sw				0	0
mount -a

# set hostname
echo <awesome hostname> > /etc/hostname

# find timezone by looking through /usr/share/zoneinfo and then run the following command
ln -s /usr/share/zoneinfo/<timezone> /etc/localtime

# install grub (omit no-nvram option if this is only installation, otherwise make sure refind is installed)
grub-install --efi-directory=/boot/efi --no-nvram

# make sure a grub.cfg is present for boot
# useful options to put on the kernel command line
#    ipv6.disable=1             disable ipv6
#    resume=UUID=<swap uuid>    for hibernation
#    consoleblank=300           blank console after 5 minutes
# add these by editing /etc/default/grub and adding options to the end of GRUB_CMDLINE_LINUX_DEFAULT
update-grub

# wrap up chroot install (also generates initrd)
xbps-reconfigure -fa
reboot
```

## after reboot

```
# login as root to enable services
ln -s /etc/sv/bluetooth /var/service
ln -s /etc/sv/dbus /var/service
ln -s /etc/sv/dhcpcd /var/service
ln -s /etc/sv/elogind /var/service
ln -s /etc/sv/iwd /var/service
ln -s /etc/sv/ntpd /var/service

# logout and login as user
cd <path of this repo>
cp -r dots/.fbtermrc dots/.gitconfig dots/.ssh ~/
mkdir ~/.config
cp -r alacritty niri tmux waybar wob xdg-desktop-portal ~/.config
cp -r local ~/

# patch bash support files
cd
patch < <path of this repo>/bash_logout.patch
patch < <path of this repo>/bash_profile.patch
patch < <path of this repo>/bashrc.patch

# setup pipewire
mkdir -p .config/pipewire/pipewire.conf.d
cd .config/pipewire/pipewire.conf.d
ln -s /usr/share/examples/wireplumber/10-wireplumber.conf
ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf

# setup and install flatpaks
flatpak --user remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak --user remote-add --if-not-exists flathub-beta https://flathub.org/beta-repo/flathub-beta.flatpakrepo
for app in $(cat flatpak_apps.txt) ; do flatpak install flathub -y --noninteractive "$app" ; done

# set codium to use sandboxed utils
flatpak --user override --env="FLATPAK_ENABLE_SDK_EXT=llvm20,node22,openjdk21" com.vscodium.codium

# logout and back in as user
start_niri
```
