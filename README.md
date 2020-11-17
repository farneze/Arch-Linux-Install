# **Arch Install**

## **System used**

- Thinkpad T440p
- Processor: i7 4700mq
- Memory: 16 gb DDR3
- SSD: HP 500GB S700

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

And partitions set for EFI boot ([Swap size](https://docs.fedoraproject.org/en-US/Fedora/13/html/Installation_Guide/s2-diskpartrecommend-x86.html]))
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

Update mirrors and download wget

```console
# pacman -Syy
# pacman -S wget
```

## **Install Arch**

Base install (btrfs related -> btrfs-progs snapper)

```console
# pacstrap /mnt base base-devel btrfs-progs snapper linux linux-firmware
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

Starting alias

```console
# alias pac="sudo pacman -S --noconfirm --needed"
```

Install kernel, micro code, sudo and other shell helpers

```console
# pac intel-ucode zsh zsh-doc zsh-completions vi vim nano wget
```

If file system is btrfs, also install

```console
# pac btrfs-progs
```

Install network

```console
# pac iwd openssh dhcpcd
```

Set time zone

```console
# ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
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
# nano /etc/hosts
-------------------hosts------------------
127.0.0.1   localhost
::1   localhost
127.0.1.1   machinename.localdomain machinename
------------------------------------------
```

Add user

```console
# useradd -m -g users -G wheel,storage -s /bin/zsh arnthor -p arnthor
```

Change default editor to nano (default: vi)

```console
# export EDITOR=nano
```

Configure users. Uncomment `#wheel` line and add below it `Defaults rootpw`

```console
# nano /etc/sudoers
------------------sudoers-----------------
...
%wheel ALL=(ALL) ALL
Defaults rootpw
...
------------------------------------------
```


## **Initial ram disk**

Edit mkinitcpio, and under `#HOOKS`:

- if using laptop with external keyboard, place `keyboard` before `autodetect`
- remove `udev` and insert `systemd`
(This [post](https://bbs.archlinux.org/viewtopic.php?id=170082) or this [one](https://bbs.archlinux.org/viewtopic.php?id=169988) confirms it)
- if using `btrfs` add it after `systemd` (or `udev` if you didnt add systemd)
- for early KMS, check this [link](https://wiki.archlinux.org/index.php/Kernel_mode_setting#Early_KMS_start)

And under `#COMPRESSION`:
 - uncomment LZ4. [link](https://www.dummeraugust.com/main/content/blog/posts.php?pid=173)

```console
# nano /etc/mkinitcpio.conf
--------------mkinitcpio.conf-------------
...
MODULES=()
...
HOOKS=(base systemd btrfs keyboard autodetect modconf block filesystems fsck)`
...
#COMPRESSION="lzop"
COMPRESSION="lz4"
#COMPRESSION="zstd"
...
------------------------------------------
```

Generate initramfs

```console
# mkinitcpio -p linux
```

## **Grub OR ...**

Install

```console
# pac  grub efibootmgr
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# grub-mkconfig -o /boot/grub/grub.cfg
```

Edit grub timeout to 0

```console
# nano /etc/default/grub
GRUB_TIMEOUT=0
```

And run `mkconfig` again

```console
# grub-mkconfig -o /boot/grub/grub.cfg
```

## **... OR systemd-boot**

Install

```console
# bootctl install
```

Add systemd-boot necessary files:
```console
# nano /boot/loader/entries/arch.conf
-----------------arch.conf----------------
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root="LABEL=arch_os" rw
options sysrq_always_enabled=1
------------------------------------------
```

Add above ???
```console
    options zswap.enabled=1 zswap.compressor=lz4
```

```console
# nano /boot/loader/loader.conf
----------------loader.conf---------------
timeout 0
# default a5e5f7c9baf5e4....-*
default  arch.conf
editor   no
------------------------------------------
```

## **Reboot**

Exit, umount any partitions and reboot

```console
# exit
# umount -R /mnt
# reboot
```

# **OS Config**

## **Thinkpad T440p keyboard**

Brazilian keyboards have an AltGr key, so that key was added to a `br-abnt2.map.gz` file and named `br-abnt2-thinkpad.map.gz`

Outside of X11:

```console
$ cp br-abnt2-thinkpad.map.gz /usr/share/kbd/keymaps/i386/qwerty
$ loadkeys /usr/share/kbd/keymaps/i386/qwerty/br-abnt2-thinkpad.map
```

For X11, add:
```console
# sudo nano ~/.xinitrc
-----------------.xinitrc-----------------
setxkbmap -model abnt2 -layout br
...
------------------------------------------
```

## **Pacman**
Pacman config:
- uncomment `Color`
- add `ILoveCandy` (optional)
- uncomment [multilib] and `include` line

```console
# sudo nano /etc/pacman.conf
----------------pacman.conf---------------
...
# Misc options
#UseSyslog
Color
#TotalDownload
CheckSpace
#VerbosePkgLists
ILoveCandy
...
[multilib]
Include = /etc/pacman.d/mirrorlist
...
------------------------------------------
```

## **Start and enable services**

```console
$ sudo systemctl enable --now iwd.service
$ sudo systemctl enable --now sshd.service
$ sudo systemctl enable --now systemd-networkd.service
```

## **Connect**

Again...

```console
$ iwctl station wlan0 connect MyWifiIsThis --passphrase PaSsPhRaSe
```

## **Update system**

```console
$ sudo pacman -Syyu
```

## **Update Mirrors**

Manually:

```console
$ wget -O mirrorlist.b "https://archlinux.org/mirrorlist/?country=BR"
$ sed -i 's/^#//' mirrorlist.b
$ sudo cp -r /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bck
$ sudo rankmirrors -n 8 mirrorlist.b > /etc/pacman.d/mirrorlist
```

Reflector:

```console
$ pac reflector
```

```console
$ sudo nano /etc/xdg/reflector/reflector.conf
-------------reflector.conf---------------
--save /etc/pacman.d/mirrorlist
--country Brazil,"United States"
--protocol https
--latest 10
--sort age
------------------------------------------
$ sudo systemctl enable reflector.service --now
```

# **Packages**

From here, there will be pacman and AUR-yay packages

### **Installing YAY - AUR Helper**

```console
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -sirc
$ alias yay="yay -S --needed"
```

### **Installs**

Complementary install

```console
$ pac man-db man-pages mesa lib32-mesa xf86-video-intel pacutils pacman-contrib ntfs-3g exfat-utils xclip
```

Main programs

```console
$ pac deluge deluge-gtk steam mpv youtube-dl telegram-desktop discord virtualbox keepassxc (chromium?)
$ yay skype zoom slack keybase-bin (google-chrome?)
```

Fonts

```console
$ pac ttf-liberation noto-fonts ttf-hack ttf-font-awesome ttf-roboto wqy-zenhei ttf-joypixels
$ yay nerd-fonts-mononoki ttf-mononoki
```

Terminal tools

```console
$ pac htop tree lsof mc bat termite
$ yey tty-clock
```

Archiving and Compression

```console
$ pac unzip unrar lha
```

Hardware tools

```console
$ pac lshw dmidecode lm_sensors acpi sensors-detect hdparm
$ xow-git
```

Dev tools

```console
$ pac git hub code neovim python nodejs npm
```

Office

```console
$ pac libreoffice-fresh
```

Media

```console
$ pac playerctl
```

Thinkpad install

```console
$ pac tpacpi-bat xf86-input-synaptics
$ yay upd72020x-fw thinkfan
```

Video editor

```console
$ pac obs-studio libva-intel-driver(?)
```

Wallpaper tools

```console
$ pac python-pywal
```

Retro Geming

```console
$ pac retroarch retroarch-assests-xmb retroarch-assests-ozone
```

Playful programs

```console
$ pac cmatrix neofetch toilet figlet lolcat nyancat cowsay ponysay asciiquarium
$ yay pipes.sh
```

Other Tools

```console
$ pac pccze plowshare net-tools mtr traceroute dnsutils whois nmap wavemon gnome-nettool sshfs proxychains-ng powerpill imagemagick ffmpeg gthumb
$ yay dtrx pcmanfm-gtk3-git redshift-gtk-git raccoon
```

# **Post-Install**

## **Nano syntax highlighting**

Install

```console
$ pac nano-syntax-highlighting
```

Uncomment below

```console
$ sudo nano /etc/nanorc
-------------reflector.conf---------------
include "/usr/share/nano/*.nanorc"
include "/usr/share/nano-syntax-highlighting/*.nanorc"
------------------------------------------
```

## **Auto-Login**

NOT RECOMMENDED
```console
$ sudo systemctl edit getty@tty1
---------------getty@tty1-----------------
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin arnthor --noclear %I $TERM
------------------------------------------
```

## **Install DE or WM**

### **KDE OR...**

```console
$ pac xorg plasma-meta kde-application-meta
$ sudo systemctl enable ssdm.service --now
```

### **...OR DWM**

```console
$ pac xorg-server xorg-xinit xorg-xrandr xorg-xsetroot pcmanfm
$ ./dmenuinstall.sh
$ ./dwminstall.sh
```

Add `exe dwm` to `.xinitrc`
```console
$ sudo nano /etc/.xinitrc
----------------.xinitrc------------------
exec dwm
------------------------------------------
```

## **Sound**

### **ALSA**

```console
$ pac alsa-utils
$ amixer -c 1 sset Speaker unmute
$ amixer -c 1 sset Headphones unmute
```

Unmute channels through alsamixer or commands:

```console
$ amixer -c 1 sset Master unmute
$ amixer -c 1 sset Speaker unmute
$ amixer -c 1 sset Headphones unmute
$ amixer -c 1 sset Mic unmute
```

If more than 2 cards, select the default:

```console
$ sudo nano /etc/asound.conf
---------------asound.conf----------------
defaults.pcm.card 1
defaults.ctl.card 1
------------------------------------------
```

To unmute MIC, open alsamixer, press F3 and SPACE.

### **PulseAudio** (optional)

```console
$ pac pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-equalizer pulseaudio-jack pavucontrol
```

### **Other packages**

```console
$ pac
$ yey xava
```

## **Screen Tearing** (optional)

Add this file

```console
$ sudo nano /etc/X11/xorg.conf.d/20-intel.conf
--------------20-intel.conf---------------
Section "Device"
    Identifier "Intel Graphics"
    Driver "intel"
    Option "DRI" "3"
    Option "AccelMethod" "sna"
    Option "TearFree" "true"
EndSection
------------------------------------------
```

## **Screen turning off** (optional)

Add this file

```console
$ sudo nano /etc/X11/xorg.conf.d/30-monitor.conf
-------------30-monitor.conf--------------
Section "Monitor"
    Identifier "LG-UW"
    Option "DPMS" "false"
EndSection

Section "ServerFlags"
    Option "StandbyTime" "0"
    Option "SuspendTime" "0"
    Option "OffTime" "0"
    Option "BlankTime" "0"
EndSection

Section "ServerLayout"
    Identifier "ServerLayout0"
EndSection
------------------------------------------
```

## **Thinkpad**


# **Cleaning**

```console
# pacman -Rsn $(pacman -Qdtq)
# pac -Sc
```

# **Theming**

Nord theme for grep:

```console
# export GREP_COLORS='ms=01;38;2;136;192;208'
```

# **Pacman commands**

List all foreign packages (typically manually downloaded and installed or packages removed from the repositories):

```console
# pacman -Qm
```

List all explicitly installed native packages (i.e. present in the sync database) that are not direct or optional dependencies:

```console
# pacman -Qent
```

# **Wallpapers**

```console
# pac archlinux-wallpaper nitrogen feh
```

# **Aliases**

Check .dot files

# **To do / study**

- [ ] KVM
- [ ] minimize initramfs
- [ ] initramfs compression
- [ ] Encrypt partitions
- [ ] Automate (partially) install
- [ ] Test zswap compression
