# **Arch Install**

## **System used**

- Thinkpad T440p
- Processor: i7 4700mq
- Memory: 16 gb DDR3
- another test

## **Known errors**

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

### **Disclaimer**

This is intended to be a simplified version of [Arch Installation Guide](https://wiki.archlinux.org/index.php/installation_guide), focused on my notebook and personal preferences. Feel free to contact me to give suggestions and help me (and possible other fellows) on new features or a better way to do something.

## **Steps**

### Arch boot image

First download [Arch Linux ISO file](https://www.archlinux.org/download/).

#### Windows

Install [rufus](https://rufus.ie/)
Open the program, select USB drive, select ISO. select GPT on Partition Scheme. Click start and select DD Image mode.

#### Linux

Create bootable USB drive using:

```console
# sudo dd if=image.iso of=/dev/penpen bs=16M && sync
```

## **First boot**

When booted, select:

> Arch Linux install medium

Connect to wifi

```console
# iwctl station wlan0 connect MyWifiIsThis --passphrase PaSsPhRaSe
```

List devices

```console
# lsblk
```

Erase disk

```console
# dd if=/dev/zero of=/dev/sda bs=16M count=1
```

Tool for defining partitions:

```console
# cfdisk
```

And partitions set for EFI boot
| Partition | Size      | Type             |
| --------- |----------:| :----------------|
| sda1      | 500MB     | EFI System       |
| sda2      | 8G        | Linux SWAP       |
| sda3      | **Rest**G | Linux filesystem |

Format and mount partions

- Linux filesystem

```console
# mkfs.btrfs -L "arch_os" /dev/sda3`
# mount -o defaults,relatime,discard,ssd /dev/sda3 /mnt
```

- Linux SWAP

```console
# mkswap -L "swap" /dev/sda2
swapon /dev/sda2
```

- EFI System

```console
# mkfs.fat -F32 /dev/sda1
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

## **Initial Configuration**

Setting time:

```console
# timedatectl set-timezone America/Sao_Paulo
# timedatectl set-ntp true
```

Packages:

```console
# pacman -S wget git
```

Brazilian mirrors:

```console
# wget -O mirrorlist.b "https://archlinux.org/mirrorlist/?country=BR"
# sed -i 's/^#//' mirrorlist.b
# rankmirrors -n 8 mirrorlist.b > /etc/pacman.d/mirrorlist
```

Pacman config:

```console
# sudo nano /etc/pacman.conf
```

- uncomment `color`
- add `ILoveCandy`
- uncomment [multilib] and `include` line

## **Install Arch**

Base install

```console
# pacstrap /mnt base base-devel linux linux-firmware
```

Copy partition tables to partition

```console
# genfstab -U /mnt >> /mnt/etc/fstab
```

Change root to new system

```console
# arch-chroot /mnt
```

## **Configure Arch**

Install kernel, micro code, sudo and other shell helpers

```console
# pacman -S intel-ucode sudo zsh zsh-doc vi vim nano curl wget
```

If file system is btrfs, also install

```console
# pacman -S btrfs-progs snapper
```

Install network

```console
# pacman -S iwd dhcpcd openssh
```

Set locale (UTF)

```console
# nano /etc/locale.gen
# locale-gen
```

Create `locale.conf` file

```console
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Create keyboard map

```console
# echo "KEYMAP=de-latin1" > /etc/vconsole.conf
```

Set Root password

```console
# passwd
```

Set time zone

```console
# ln -sf /usr/share/zoneinfo/$(curl https://ipapi.co/timezone) /etc/localtime
```

Adjust clock

```console
# hwclock --systohc
```

Set machine name

```console
# echo "machinename" > /etc/hostname
```

Set `localhost` file to `/etc/hosts`

```console
# echo "127.0.0.1   localhost" >> /etc/hosts
# echo "::1   localhost" >> /etc/hosts
# echo "127.0.1.1   machinename.localdomain machinename" >> /etc/hosts
```

Add user

```console
# useradd -m -g users -G wheel,storage -s /bin/zsh arnthor passwd arnthor
```

Configure users. Uncomment `#wheel` line and add below it `Defaults rootpw`

Command:

```console
# EDITOR=nano visudo
```

Text:
> %wheel ALL=(ALL) ALL
>
> Defaults rootpw

## **Initial ram disk**

Edit mkinitcpio

```console
# nano /etc/mkinitcpio.conf
```

Under `#HOOKS`

- if using `btrfs` add it after `udev`
- if using laptop with external keyboard, place `keyboard` before `autodetect`
- remove `udev` and insert `systemd`  
(This [post](https://bbs.archlinux.org/viewtopic.php?id=170082) or this [one](https://bbs.archlinux.org/viewtopic.php?id=169988) confirms it)

> HOOKS=(base systemd btrfs autodetect modconf block filesystems keyboard fsck)`

Under `#COMPRESSION`, uncomment LZ4. [link](https://www.dummeraugust.com/main/content/blog/posts.php?pid=173)

> #COMPRESSION="lzop"
>
> COMPRESSION="lz4"
>
> #COMPRESSION="zstd"

Generate initramfs

```console
# mkinitcpio -p linux
```

## **Grub OR ...**

Install

```console
# pacman -S grub efibootmgr

# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# grub-mkconfig -o /boot/grub/grub.cfg
```

Edit grub timeout to 0

```console
# nano /etc/default/grub
```

> GRUB_TIMEOUT=0

And run `mkconfig` again

```console
# grub-mkconfig -o /boot/grub/grub.cfg
```

## **... OR systemd-boot**

Install

```console
# bootctl install
```

Edit `/esp/loader/loader.conf`

> default  arch.conf
>
> editor   no

Edit `/boot/loader/entries/arch.conf`
> title Arch Linux
>
> linux /vmlinuz-linux
>
> initrd /intel-ucode.img
>
> initrd /initramfs-linux.img
>
> options root="LABEL=arch_os" rw

```console
    options sysrq_always_enabled=1
    options zswap.enabled=1 zswap.compressor=lz4
```

## **Reboot**

Exit, umount any partitions and reboot

```console
# exit
# umount -R /mnt
# reboot
```

## **Configs**

### **iwd**

Set iwd as wifi backend

```console
# nano /etc/NetworkManager/conf.d/wifi_backend.conf
```

>[device]
>
>wifi.backend=iwd

## **Update system**

```console
# pacman -Syyu
```

## **Start and enable services**

```console
# systemctl enable iwd.service --now
# systemctl enable dhcpcd.service --now
# systemctl enable sshd.service --now
```

## **KDE OR...**

Nord theme for grep:

```console
# pacman -S xorg plasma-meta kde-application-meta
# systemctl enable ssdm.service --now
```

## **...OR DWM**

Nord theme for grep:

```console
# pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xsetroot
# ./dmenuinstall.sh
# ./dwminstall.sh
```

## **Packages**

Complementary install

```console
# pacman -S man-db man-pages mesa lib32-mesa xf86-video-intel pacutils pacman-contrib ntfs-3g exfat-utils xclip
```

Fonts

```console
# pacman -S ttf-liberation noto-fonts ttf-hack ttf-font-awesome ttf-roboto wqy-zenhei wqy-microhei wqy-microhei-lite
```

Test

```console
# pacman -S iproute2 (?)
```

Terminal tools

```console
# pacman -S htop tree lsof mc
```

Archiving and Compression

```console
# pacman -S unzip unrar lha
```

Hardware tools

```console
# pacman -S lshw dmidecode lm_sensors
```

Dev tools

```console
# pacman -S git hub code python npm
```

Media

```console
# pacman -S feh playerctl imagemagick ffmpeg gthumb
```

Online Tools

```console
# pacman -S pccze plowshare
```

sudo pacman --noconfirm -S   net-tools mtr traceroute dnsutils whois nmap wavemon gnome-nettool sshfs proxychains-ng powerpill

Thinkpad install

```console
# pacman -S tpacpi-bat
```

Common programs

```console
# pacman -S deluge deluge-gtk steam alacritty mpv youtube-dl telegram discord virtualbox keepassxc
```

Terminal utilities

```console
# pacman -S bat
```

Edition programs

```console
# pacman -S obs-studio
```

Wallpaper tools

```console
# pacman -S python-pywal
```

Playful programs

```console
# pacman -S cmatrix neofetch figlet lolcat nyancat cowsay ponysay
```

## **AUR Packages**

Installing YAY - AUR Helper

```console
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -sirc
```

Install AUR Packages

```console
yay -S dtrx pcmanfm-gtk3-git ranger-git redshift-gtk-git raccoon wcalc pipes
```

## **Cleaning**

```console
# pacman -Rsn $(pacman -Qdtq)
# pacman -Sc
```

## **Theming**

Nord theme for grep:

```console
# export GREP_COLORS='ms=01;38;2;136;192;208'
```

Nord theme for LS:

```console
#
```

Nord theme for man:

Nord theme for pacman:

## Pacman commands

List all availlable packages which match **name**:

```console
pacman -Ss [ name ]
```

Installs package whith **name**:

```console
# pacman -S [ name ]
```

List all explicitly installed packages:

```console
# pacman -Qe
```

List all foreign packages (typically manually downloaded and installed or packages removed from the repositories):

```console
# pacman -Qm
```

List all explicitly installed native packages (i.e. present in the sync database) that are not direct or optional dependencies:

```console
# pacman -Qent
```

## Aliases

Check .dot files

### **To do / study**

[] KVM
- minimize initramfs
- initramfs compression
- Encrypt partitions
- Automate (partially) install
- Create bin folder with scripts
