# Install Arch Linux on aarch64 macOS

``` yaml
date: 2022-04-04
```

This guide aims to help you to install [Arch Linux ARM] into a [UTM] virtual machine
with [Alpine Linux] as the live system for installing Arch.

[Arch Linux ARM]: https://archlinuxarm.org/platforms/armv8/generic
[UTM]: https://mac.getutm.app
[Alpine Linux]: https://alpinelinux.org

## Other guides

- [Installing Arch Linux ARM into a UTM virtual machine]
- [Running Manjaro ARM in UTM on M1 Mac]

[Installing Arch Linux ARM into a UTM virtual machine]: https://ktprograms.codeberg.page/blog/posts/2022-03-17_1750_utm-arch-arm/
[Running Manjaro ARM in UTM on M1 Mac]: https://www.appelgriebsch.org/005-utm/

## Download Alpine

[Download Alpine]—Select `Standard | aarch64`.

[Download Alpine]: https://alpinelinux.org/downloads/

Verify the integrity of the ISO:

``` sh
shasum -a 256 alpine-standard-3.15.4-aarch64.iso
```

[shasum(1)](https://man.archlinux.org/man/shasum.1)

Expected output:

```
1ea9f72a04b5ed916dce3b0db629fb13f8e98eb489f11b64ceefb434b6c51a01
```

## Create the virtual machine

Open UTM.

Create a new virtual machine.

``` yaml
Name: Arch Linux
Architecture: ARM64 (aarch64)
RAM: 8 GB
Storage: 64 GB
Enable hardware OpenGL acceleration: yes
Enable directory sharing: yes
Boot image: alpine-standard-3.15.4-aarch64.iso
```

Click the play button.

## Set up Alpine

Log in as `root`. No password is required.

``` sh
service hwclock start
setup-alpine
```

See [setup-alpine] for details.

[setup-alpine]: https://docs.alpinelinux.org/user-handbook/0.1a/Installing/setup_alpine.html

## Continue install via SSH (optional)

This step is optional, to compensate for the lack of clipboard sharing.

``` sh
sed -i '1iPermitRootLogin yes' /etc/ssh/sshd_config
service sshd restart
ifconfig | grep 192.168 # 192.168.64.3
```

[sed(1)](https://man.archlinux.org/man/sed.1),
[sshd_config(5)](https://man.openbsd.org/sshd_config)

Open your favorite terminal emulator to ssh Alpine.

``` sh
ssh root@192.168.64.3
```

You can now copy/paste commands from your host—macOS—system.

## Install needed packages

``` sh
apk add parted curl e2fsprogs libarchive-tools efibootmgr
```

See [Alpine Linux packages] for details.

[Alpine Linux packages]: https://pkgs.alpinelinux.org

## Other install procedures

The chapter [Installing NixOS] has great and up to date examples for partitioning and formatting the system,
if you want to do things differently.

[Installing NixOS]: https://nixos.org/manual/nixos/stable/index.html#sec-installation

## Partition the disk

``` sh
parted /dev/vda -- mklabel gpt
parted /dev/vda -- mkpart boot 1MiB 512MiB
parted /dev/vda -- mkpart root 512MiB 100%
parted /dev/vda -- set 1 esp on
parted /dev/vda -- print
```

[parted(8)](https://man.archlinux.org/man/parted.8)

For details on parted subcommands, see [Parted Session Commands].

[Parted Session Commands]: https://gnu.org/software/parted/manual/parted.html#Parted-Session-Commands

## Format the filesystems

``` sh
mkfs.vfat -F 32 -n boot /dev/vda1
mkfs.ext4 -L root /dev/vda2
```

[mkfs.fat(8)](https://man.archlinux.org/man/mkfs.fat.8),
[mkfs.ext4(8)](https://man.archlinux.org/man/mkfs.ext4.8)

## Mount the filesystems

``` sh
mount -t ext4 /dev/vda2 /mnt
mkdir /mnt/boot
mount -t vfat /dev/vda1 /mnt/boot
```

[mount(8)](https://man.archlinux.org/man/mount.8)

## Download the latest—[Arch Linux ARM]—tarball and write it to the disk

``` sh
curl -L http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz | bsdtar -xp -C /mnt
```

[curl(1)](https://man.archlinux.org/man/curl.1),
[bsdtar(1)](https://man.archlinux.org/man/bsdtar.1)

## Update fstab

``` sh
echo /dev/vda1 /boot vfat defaults 0 0 >> /mnt/etc/fstab
```

[fstab(5)](https://man.archlinux.org/man/fstab.5)

## Add an “Arch Linux” boot entry

The Gentoo page—[efibootmgr(Gentoo wiki)]—has good examples for managing boot entries,
if you want to explore efibootmgr’s capabilities.

[efibootmgr(Gentoo wiki)]: https://wiki.gentoo.org/wiki/Efibootmgr

``` sh
efibootmgr -c -d /dev/vda -p 1 -L 'Arch Linux' -l '\Image' -u 'console=tty1 quiet root=/dev/vda2 rw initrd=\initramfs-linux.img'
```

[efibootmgr(8)](https://man.archlinux.org/man/efibootmgr.8)

For details on kernel parameters, see [The EFI Boot Stub].

[The EFI Boot Stub]: https://docs.kernel.org/admin-guide/efi-stub.html

## Shut down the VM and remove the ISO

``` sh
poweroff
```

[poweroff(8)](https://man.archlinux.org/man/poweroff.8)

Clear CD/DVD.

## Boot “Arch Linux” and run post-installation tasks

Log in as `root` with the password `root` and run the [post-installation tasks][Arch Linux ARM].

``` sh
pacman-key --init
pacman-key --populate archlinuxarm
pacman -Syu
```

[pacman-key(8)](https://man.archlinux.org/man/pacman-key.8),
[pacman(8)](https://man.archlinux.org/man/pacman.8)

## Install the SPICE guest additions

``` sh
pacman -S spice-vdagent
```

## Install the Yaourt AUR helper

[Install paru]—or another AUR helper.

``` sh
pacman -S base-devel git
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

[makepkg(8)](https://man.archlinux.org/man/makepkg.8)

[Install paru]: https://github.com/Morganamilo/paru#installation

## Configure the system

At this point, you can follow the rest of the [Arch Linux installation guide] to configure the system.

Time zone:

``` sh
ln -s /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock -w
```

[hwclock(8)](https://man.archlinux.org/man/hwclock.8)

Localization:

``` sh
echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

[locale.conf(5)](https://man.archlinux.org/man/locale.conf.5)

Network configuration:

``` sh
echo kanto > /etc/hostname
```

[hostname(5)](https://man.archlinux.org/man/hostname.5)

Root password—the default password is “root”:

``` sh
passwd
```

[passwd(1)](https://man.archlinux.org/man/passwd.1)

Finally, add a new user for daily use.

``` sh
useradd -m -G wheel,users taupiqueur
passwd taupiqueur
echo '%wheel ALL=(ALL:ALL) ALL' >> /etc/sudoers
```

[useradd(8)](https://man.archlinux.org/man/useradd.8)

[Arch Linux installation guide]: https://wiki.archlinux.org/title/Installation_guide
