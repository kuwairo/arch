### Arch Linux (EFISTUB, Btrfs, zram, GNOME, Wayland)

#### WiFi

```
# iwctl
[iwd]# device list
[iwd]# station <device-name> scan
[iwd]# station <device-name> get-networks
[iwd]# station <device-name> connect <network-name>
Passphrase: <...>
[iwd]# station <device-name> show
[iwd]# exit
```

#### NTP

```
# timedatectl set-ntp true
```

#### Partitions

UEFI with GPT

| Mount point      | Partition     | Type                 | Size    |
|------------------|---------------|----------------------|---------|
| `/mnt/boot`      | `/dev/<efi>`  | EFI system partition | 512 MiB |
| `/mnt` (for now) | `/dev/<root>` | Linux filesystem     | <...>   |

```
# lsblk
# cfdisk
```

```
# mkfs.fat -F 32 /dev/<efi>
# mkfs.btrfs /dev/<root>
```

```
# mount /dev/<root> /mnt
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@home
```

```
# umount /mnt
# mount -o noatime,compress=zstd,subvol=@ /dev/<root> /mnt
```

```
# mkdir /mnt/{boot,home}
# mount /dev/<efi> /mnt/boot
# mount -o noatime,compress=zstd,subvol=@home /dev/<root> /mnt/home
```

#### Essential packages

```
# pacstrap /mnt \
    base \
    base-devel \
    linux \
    linux-headers \
    linux-firmware \
    btrfs-progs \
    intel-ucode \
    bash-completion \
    man-db \
    man-pages \
    vim
```

#### Fstab

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

#### Chroot

```
# arch-chroot /mnt
```

#### Basic setup

```
# ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
# hwclock --systohc
# # uncomment 'en_US.UTF-8 UTF-8' in /etc/locale.gen
# locale-gen
# echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

#### TRIM

```
# systemctl enable fstrim.timer
```

#### Network configuration

```
# echo "<hostname>" >> /etc/hostname
# echo "\
> 127.0.0.1  localhost
> ::1        localhost
> 127.0.1.1  <hostname>.localdomain <hostname>" >> /etc/hosts
```

```
# pacman -S networkmanager firewalld openssh rsync
# systemctl enable NetworkManager.service
# systemctl enable firewalld.service
```

#### Reflector

```
# pacman -S reflector
# mv /etc/pacman.d/mirrorlist \
     /etc/pacman.d/mirrorlist.bkp
```

```
# echo "\
> --country <C1,C2,...,Cn>
> --latest 10
> --sort rate
> --protocol https
> --save /etc/pacman.d/mirrorlist" >> /etc/xdg/reflector/reflector.conf
```

```
# systemctl enable reflector.timer
# systemctl start reflector.service
```

#### Bluetooth

```
# pacman -S bluez bluez-utils
# systemctl enable bluetooth.service
```

#### Power management

```
# pacman -S tlp tlp-rdw
# systemctl enable tlp.service
# systemctl enable NetworkManager-dispatcher.service
# systemctl mask systemd-rfkill.service
# systemctl mask systemd-rfkill.socket
```

#### zram

```
# pacman -S zram-generator
# echo "\
> [zram0]
> zram-size = min(ram / 2, 8192)" >> /etc/systemd/zram-generator.conf
```

#### Audio

```
# pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack
```

#### Video

```
# pacman -S mesa intel-media-driver libva-utils mpv
# # If vainfo fails (after reboot), set LIBVA_DRIVER_NAME=iHD (e.g. in /etc/environment)
# # In Firefox (about:config), set media.ffmpeg.vaapi.enabled = true
```

#### GNOME

```
# pacman -S gnome gnome-tweaks gnome-themes-extra \
            firefox papirus-icon-theme xdg-user-dirs
# systemctl enable gdm.service
```

#### Printer

```
# pacman -S cups cups-pdf
# systemctl enable cups.service
```

#### User

```
# passwd
# useradd -m -G wheel <username>
# passwd <username>
# visudo # uncomment '%wheel ALL=(ALL:ALL) ALL'
```

#### EFISTUB

```
# pacman -S efibootmgr
# blkid # to find PARTUUID
# efibootmgr -d /dev/<boot-device> -p <boot-part-no> -c -L "Arch Linux" -l /vmlinuz-linux \
-u 'root=PARTUUID=<root-PARTUUID> rootflags=noatime,compress=zstd,subvol=@ rw initrd=\intel-ucode.img initrd=\initramfs-linux.img'
```

#### Initramfs

```
# mkinitcpio -p linux
```

#### Reboot

```
# exit
# umount -R /mnt
# reboot
```

