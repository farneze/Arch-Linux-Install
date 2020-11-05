# Arch Install

## System used

- Thinkpad T440p
  - Processor: i7 4700mq
  - Memory: 16 gb DDR3
  - another test

## Known errors

```console
==> WARNING: Possibly missing firmware for module: aic94xx
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: xhci_pci
```

Ignore the first two errors, as those are related to SCSI drivers for devices we will never use. On this [link](https://gist.github.com/imrvelj/c65cd5ca7f5505a65e59204f5a3f7a6d), there is a fix from AUR packages, but `rigred` comments:

> Of importance is that this module is entirely unnecessary for most users.

After some search, the hardware that uses those module are:

- aic94xx => Adaptec Serial Attached SCSI
- wd719x => Driver for Western Digital SCSI cards

So, just ignore those errors.

The last one (`xhci_pci`) appears to me because I have a device with a Renesas USB Controller, a Lenovo Docking Station for my Thinkpad T440p.

To fix it, ***as user*** and ***not as root***, install [this](https://aur.archlinux.org/packages/upd72020x-fw/) AUR package after initial install and reboot.

---

## Disclaimer

This is intended to be a simplified version of [Arch Installation Guide](https://wiki.archlinux.org/index.php/installation_guide), focused on my notebook and personal preferences. Feel free to contact me to give suggestions and help me (and possible other fellows) on new features or a better way to do something.

---

## Steps

### Arch boot image

First download [Arch Linux ISO file](https://www.archlinux.org/download/).

#### Windows

Install [rufus](https://rufus.ie/)

#### Linux

> _#_ `tar -xzOf archlinux-YYYY-MM-DD.iso | dd of=/dev/sdb1 bs=4M status=progress && sync`

---

## First boot

When booted, select:

> Arch Linux install medium

Connect to wifi

> _#_ `iwctl station wlan0 connect MyWifiIsThis --passphrase PaSsPhRaSe`

List devices
> _#_ `lsblk`

Erase disk
> _#_ `dd if=/dev/zero of=/dev/sda bs=16M count=1`

Tool for defining partitions:
> _#_ `cfdisk`

And partitions set for EFI boot
| Partition | Size    | Type             |
| --------- |--------:| :----------------|
| sda1      | 500MB   | EFI System       |
| sda2      | 8G      | Linux SWAP       |
| sda3      | `Rest`G | Linux filesystem |

Format and mount partions

- Linux filesystem

    > _#_ `mkfs.btrfs -L "Arch Linux" /dev/sda3`
    >
    > _#_ `mount -o defaults,relatime,discard,ssd /dev/sda3 /mnt`

- Linux SWAP

    > _#_ `mkswap -L "swap" /dev/sda2`
    >
    > _#_ `swapon /dev/sda2`

- EFI System

    > _#_ `mkfs.fat -F32 /dev/sda1`
    >
    > _#_ `mkdir /mnt/boot`
    >
    > _#_ `mount /dev/sda1 /mnt/boot`

---

## Initial Configuration

Setting time:

> _#_ `timedatectl set-timezone America/Sao_Paulo`
>
> _#_ `timedatectl set-ntp true`

Packages:
> _#_ `pacman -S wget git`

Brazilian mirrors:
> _#_ `wget -O mirrorlist.b "https://archlinux.org/mirrorlist/?country=BR"`
>
> _#_ `sed -i 's/^#//' mirrorlist.b`
`rankmirrors -n 8 mirrorlist.b > /etc/pacman.d/mirrorlist`

Pacman config:

On `sudo nano /etc/pacman.conf`

- uncomment `color`
- add `ILoveCandy`
- uncomment [multilib] and `include` line

---

## Install Arch

Base install
> _#_ `pacstrap /mnt base base-devel`

Copy partition tables to partition
> _#_ `genfstab -U /mnt >> /mnt/etc/fstab`

Change root to new system
> _#_ `arch-chroot /mnt`

---

## Configure Arch

Set locale (UTF)
> _#_ `nano /etc/locale.gen`
>
> _#_ `locale-gen`

Create `locale.conf` file
> _#_ `echo "LANG=en_US.UTF-8" > /etc/locale.conf`

Create keyboard map
> _#_ `echo "KEYMAP=de-latin1" > /etc/vconsole.conf`

Set Root password
> _$_ `passwd`

Set time zone
> _#_ `ln -sf /usr/share/zoneinfo/$(curl https://ipapi.co/timezone) /etc/localtime`

Adjust clock
> _#_ `hwclock --systohc`

Set machine name
> _#_ `echo "machinename" > /etc/hostname`

Set `localhost` file to `/etc/hosts`
> _#_ `echo "127.0.0.1   localhost" >> /etc/hosts`
>
> _#_ `echo "::1   localhost" >> /etc/hosts`
>
> _#_ `echo "127.0.1.1   machinename.localdomain machinename" >> /etc/hosts`

Add user
> _#_ `useradd -m -g users -G wheel,storage,power -s /bin/zsh arnthor passwd arnthor`

---

## mkinitcpio

Edit mkinitcpio
> _#_ `nano /etc/mkinitcpio.conf`

Under `#HOOKS`

- if using `btrfs` add it after `udev`
- if using laptop with external keyboard, place `keyboard` before `autodetect`

> _#_ `HOOKS=(base udev btrfs keyboard autodetect modconf block filesystems fsck)`


Under `#COMPRESSION`, uncomment LZ4. [link](https://www.dummeraugust.com/main/content/blog/posts.php?pid=173)
> _#_ `COMPRESSION="lz4"`


mkinitcpio -p linux

Install bootloader
> _#_ `pacman -S linux linux-firmware grub efibootmgr intel-ucode sudo`

---

## Packages

Complementary install
> _#_ `pacman -S man-db man-pages mesa lib32-mesa xf86-video-intel pacutils pacman-contrib ntfs-3g`

Btrfs packages
> _#_ `btrfs-progs snapper`

Terminal utilities
> _#_ `pacman -S iwd dhcpcd zsh zsh-doc vi vim nano lynx curl bat wget`

Battery info
> _#_ `pacman -S tpacpi-bat`

Fonts
> _#_ `pacman -S ttf-liberation`

Test
> _#_ `pacman -S iproute2 (?)`

---

## GRUB

Install
> _#_ `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
>
> _#_ `grub-mkconfig -o /boot/grub/grub.cfg`

---

## Edit mkinitcpio

Exit, umount any partitions and reboot
> _#_ `exit`
>
> _#_ `umount -R /mnt`
>
> _#_ `reboot`

---


## Configs

- iwd

    /etc/NetworkManager/conf.d/wifi_backend.conf

    [device]
    wifi.backend=iwd


---

## Other Packages

Terminal tools
> _#_ `pacman -S htop tree openssh`

Dev tools
> _#_ `pacman -S git code python npm`

Thinkpad install
> _#_ `pacman -S`

Common programs
> _#_ `pacman -S deluge deluge-gtk steam alacritty mpv youtube-dl telegram discord virtualbox keepassxc`

Edition programs
> _#_ `pacman -S obs-studio`

Playful programs
> _#_ `pacman -S cmatrix neofetch figlet lolcat nyancat cowsay ponysay`

##### For programs who asked, i used
###### Steam 3 - 3
###### virtualbox 2

---

## AUR Packages

---

## Cleaning

> _#_ `sudo pacman -Rsn $(pacman -Qdtq)`
> _#_ `sudo pacman -Sc`

---

## Start and enable services

> _#_ `systemctl enable iwd.service --now`
> _#_ `systemctl enable dhcpd.service --now`
> _#_ `systemctl enable sshd.service --now`
> _#_ `systemctl enable ssdm.service --now`

---

## Theming

Nord theme for grep:
> _#_ `export GREP_COLORS='ms=01;38;2;136;192;208'`

Nord theme for LS:
> _#_ ``

Nord theme for man:
> _#_ ``

Nord theme for pacman:

> _#_ ``

---

## Pacman commands

List all availlable packages which match **name**:
> _#_ `pacman -Ss` [**name**]

Installs package whith **name**:
> _#_ `pacman -S` [**name**]

List all explicitly installed packages:
> _#_ `pacman -Qe`

List all foreign packages (typically manually downloaded and installed or packages removed from the repositories):
> _#_ `pacman -Qm`

List all explicitly installed native packages (i.e. present in the sync database) that are not direct or optional dependencies:
> _#_ `pacman -Qent`

---

## Aliases

> _#_ `alias pacman='sudo pacman -S --needed'`
>
> _#_ `alias grep='grep --color=auto'`
>
> _#_ `alias cat='bat'`
>
> _#_ `alias mounte='mount -o gid=users,fmask=113,dmask=002 /dev/sdb1 /mnt/external/'`
>
> _#_ `alias umounte='umount /mnt/external/'`
>
> _#_ `alias weather='curl http://wttr.in/sao_paulo'`

---

### To do / study

- KVM
- minimize initramfs
- initramfs compression
