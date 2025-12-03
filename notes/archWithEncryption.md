```sh
##########################################
$ timedatectl set-ntp true

##########################################
# set partitions
$ cfdisk /dev/nvme0n1

##########################################
# format partitions
$ mkfs.fat -F32 /dev/nvme0n1p1

$ cryptsetup luksFormat /dev/nvme0n1p2
$ cryptsetup open /dev/nvme0n1p2 cryptroot
$ mkfs.ext4 /dev/mapper/cryptroot

$ cryptsetup luksFormat /dev/nvme0n1p3
$ cryptsetup open /dev/nvme0n1p3 crypthome
$ mkfs.ext4 /dev/mapper/crypthome

##########################################
# mount partitions
$ mount /dev/mapper/cryptroot /mnt

$ mkdir -p /mnt/boot/
$ mount /dev/nvme0n1p1 /mnt/boot/

$ mkdir -p /mnt/home
$ mount /dev/mapper/crypthome /mnt/home

##########################################
# install the system
$ pacstrap /mnt linux linux-lts linux-zen linux-headers linux-firmware base base-devel efibootmgr cryptsetup networkmanager neovim mtools dosfstools intel-ucode intel-media-driver vulkan-intel intel-gmmlib sof-firmware

##########################################
# generate fstab
$ genfstab -U /mnt >> /mnt/etc/fstab

##########################################
# get into the system
$ arch-chroot /mnt

##########################################
# set system time
$ ln -sf /usr/share/zoneinfo/Asia/Dhaka /etc/localtime
$ hwclock --systohc

##########################################
# locale
$ nvim /etc/locale.gen
# uncomment : en_US.UTF-8 UTF-8. write and quit vim
$ locale-gen

$ nvim /etc/locale.conf
# add the line
LANG = en_US.UTF-8

##########################################
# select hostname
$ nvim /etc/hostname

##########################################
# hosts
$ nvim /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch.localdomain  arch

##########################################
# set root password
$ passwd

# set user
$ useradd -mG wheel grimm
$ passwd grimm
$ EDITOR=nvim visudo
# uncomment the following line
%wheel All=(All) All

##########################################
# update hooks
$ nvim /etc/mkinitcpio.conf
# Find the `HOOKS` line and insert `sd-encrypt` **after** `block` and **before** `filesystems`:
$ mkinitcpio -P

##########################################
# get root and home UUID
$ blkid /dev/nvme0n1p2
$ blkid /dev/nvme0n1p3

# edit crypttab
$ nvim /etc/crypttab
crypthome       UUID=<UUID of /dev/nvme0n1p3>               none                  luks

##########################################
# install bootloader
$ bootctl --path=/boot install
$ cd /boot/loader/
$ nvim loader.conf

# add the line(s)
default arch.conf

$ cd entries/
$ touch arch.conf arch-fb.conf arch-lts.conf arch-lts-fb.conf arch-zen.conf arch-zen-fb.conf

# arch.conf
title	Arch Linux
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

# arch-fb.conf
title	Arch Linux Fallback Image
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux-fallback.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

# arch-lts.conf
title	Arch Linux LTS
linux	/vmlinuz-linux-lts
initrd	/intel-ucode.img
initrd	/initramfs-linux-lts.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

# arch-lts-fb.conf
title	Arch Linux LTS Fallback
linux	/vmlinuz-linux-lts
initrd	/intel-ucode.img
initrd	/initramfs-linux-lts-fallback.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

# arch-zen.conf
title	Arch Linux Zen
linux	/vmlinuz-linux-zen
initrd	/intel-ucode.img
initrd	/initramfs-linux-zen.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

# arch-zen-fb.conf
title	Arch Linux Zen Fallback
linux	/vmlinuz-linux-zen
initrd	/intel-ucode.img
initrd	/initramfs-linux-zen-fallback.img
options	rd.luks.name=UUID=cryptroot root=/dev/mapper/cryptroot rw

$ exit
$ umount -a
```
