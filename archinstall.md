## Archlinux installation

## keyboard setting
$ localectl list-keymaps | grep latin1
$ loadkeys <name-keymap>

## connect to wifi
```
$ iwctl
[iwctl]# device list 
wlano		# wireless interface name
[iwctl]# station <wlan0> get-networks
[iwctl]# station <wlan0> connect <"wifi-name">
[iwctl]# exit
ip addr show		# check ip address for wireless interface
$ ping -c 5 archlinux.org
```

### connect vis ssh
$ passwd 		# set root passwd
$ ssh root@<ip-addr>

### time date setup
$ timedatectl
$ timedatectl list-timezones | grep Berlin
$ timedatectl set-timezone Europe/Berlin

-----------------------------------------------------------------------------
## setup disk partition setup UEFI with "encryption" 

### check if my machine is efi or not
$ ls /sys/firmware/efi/efivars		# if content display then mac in EFI

### use fdisk /dev/partition or gdisk /dev/partition
```
$ lsblk
$ fdisk -l
$ fdisk /dev/nvme1n1
```

### Create a new partition 1 of type "EFI filesystem" and of size 500MiB 
```
$ g	# create a new GPT disklabel
$ n	# new partition
$ 1	# Partition number (default)
$ default	# First sector (default)
$ +1024M	# Last sector
$ t	# Partition type
$ 1	# partition type (1 is for EFI System)
```

### Create a new partition 2 of type "Linux filesystem" and of size 100%FREE
```
$ g       # create a new GPT disklabel
$ n       # new partition
$ 2       # Partition number (default)
$ default    # First sector (default)
$ default   # Last sector
$ t       # Partition type
$ 2	# Partition number (default)
$ 44       # partition type (44 is for LVM System)

$ p	# print partition table just created
$ w	# write partition table on the disk
```
----------------------------------------------------------------------------------------
```OUTPUT:

Device		       Start		 End		  Sectors		Size	 Type
/dev/nvme1n1p1	2048		1026047		1024000		1024M	EFI System
/dev/nvme1n1p2	2050048		1048575966	1046525919	950G	Linux LVM
```
-----------------------------------------------------------------------------------------

###To encrypt our '/' partition, we will use the cryptsetup tool:
```
cryptsetup --hash sha512 --use-random --verify-passhphrase luksFormat /dev/nvme0n1p2
Are you sure? YES
Enter passhphrase (twice)
```

Opening the root LUKS volume as block device

The / partition being encrypted, we will open the LUKS container on /dev/nvme0n1p2 disk and name it cryptlvm:

```
cryptsetup open /dev/nvme1n1p2 cryptlvm
Enter passhphrase
```
The decrypted container is now available at `/dev/mapper/cryptlvm`.

### Unlock encrypted partition
```
$ cryptsetup open --type luks /dev/nvme1n1p3 lvm
Enter passphrase for /dev/nvme1n1p3:
```

## Setup the LVM
LVM is a logical volume manager for the Linux kernel. It is thank to to it that we can easily resize our partitions if necessary.
## Create a physical volume on top of the opened LUKS container
``` $ pvcreate /dev/mapper/cryptlvm ```

### Add the previously created physical volume to a volume group
``` $ vgcreate volgroup0 /dev/mapper/cryptlvm ```

## Create the swap logical volume on the volume group

## Create the root logical volume on the volume group
``` $ lvcreate -l 100%FREE volgroup0 -n vg_root ```

## Formatting the filesystem
```
$ mkfs.fat -F32 /dev/nvme0n1p1
$ mkfs.btrfs -L btrfs /dev/mapper/vg-root

$ modprobe dm_mod
$ vgscan
$ vgchange -ay
```
------------------------------------------------------------------------------------------------
## Create Btrfs subvolumes
```
$ mount /dev/mapper/vg_root /mnt
$ btrfs subvolume create /mnt/root
$ btrfs subvolume create /mnt/home
$ umount /mnt
```

## Mounting Btrfs subvolumes
```
$ SSD_MOUNTS="autodefrag,compress=lzo,discard,noatime,nodev,rw,space_cache,ssd"
$ mount -o subvol=root,$SSD_MOUNTS /dev/mapper/vg_root /mnt

$ mkdir -p /mnt/{boot,home}

$ mount -o discard,noatime,nodev,noexec,nosuid,rw /dev/nvme1n1p1 /mnt/boot
$ mount -o $SSD_MOUNTS,nosuid,subvol=home

$ mount /dev/nvme1n1p1 /mnt/boot

$ mkfs.ext4 /dev/volgroup0/lv_home
$ mkdir /mnt/home
$ mount /dev/volgroup0/lv_home /mnt/home
```
---------------------------------------------------------------------------------------------------------

## OPTION 3: btrfs partition setup
```
$ mount /dev/nvme1n1p1 /mnt

$ cd /mnt
$ btrfs subvolume create @
$ btrfs subvolume create @home
$ cd ..
$ umount /mnt

$ mount -o noatime,ssd,space_cache=v2,compress=zstd,discard=async,subvol=@ /dev/nvme1n1p1 /mnt

$ mkdir /mnt/home
$ mount -o noatime,ssd,space_cache=v2,compress=zstd,discard=async,subvol=@home /dev/nvme1n1p2 /mnt/home
```
------------------------------------------------------------------------------------------------------------

$ reflector --threads 8 --sort rate --save --country Germany /etc/pacman.d/mirrorlist

## Installation of the packages onto a given root file system
``` $ pacstrap /mnt base linux linux-firmware linux-tools ```

## Configuration of the system
### generate fstab
```
$ mkdir /mnt/etc
$ genfstab -U -p /mnt >> /mnt/etc/fstab
$ cat /mnt/etc/fstab	# check mounted partition on fstab
```
--------------------------------------------------------------------------------------
### Change root into new system
``` $ arch-chroot /mnt ```

## Create the swap file
```
$ truncate -s 0 /swapfile
$ chattr +C /swapfile
$ btrfs property set /swapfile compression none
```

### Is recommended to set the size of swap file being 4-8GB if you don't intend to use hibernation (suspend-to-disk):
```
$ dd if=/dev/zero of=/swapfile bs=1M count=16384 status=progress && sync

###Set the permission for the file (a world-readable swap file is a huge local vulnerability):
```$ chmod 600 /swapfile ```

## Format the /swapfile file to swap and activate it:
```
$ mkswap /swapfile
$ swapon /swapfile
```

###Finally, add an entry for the swap file in the fstab:
``` $ echo "/swapfile none swap defaults 0 0" >> /etc/fstab ```
```
$ mkdir -p /mnt/boot/efi
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```
-------------------------------------------------------------------------------------------------------------

## Set the timezone

### Install the Network Time Protocol and enable it as daemon:
```
$ pacman -S ntp
$ systemctl enable ntpd
```

### Create a symlink of the timezone:
```
$ ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

### Generate /etc/adjtime:
``` $ hwclock --systohc ```

## Network configuration
### Install and enable network management daemon:
```
$ pacman -S networkmanager
$ systemctl enable NetworkManager
```

### add a hostname to the machine:
``` $ echo arch > /etc/hostname ```

## Add matching entries to /etc/hosts:
```
127.0.0.1		localhost
::1					   localhost
127.0.1.1		arch.localdomain	arch
```

## Create the main user
```
$ useradd -mG storage,wheel -s /bin/bash username
$ passwd *****
```
### Finally, change the /etc/sudoers file according to the config that you want to deal with sudo command.
```
$ EDITOR=nano visudo
%wheel ALL=(ALL) ALL	# uncomment this line
$ usermod -c 'Puneet Tomar' tomar	# show full name instead of username on login
```

$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
$ echo "KEYMAP=en_DE-latin1" >> /etc/vconsole.conf

## You should the HOOKS field with these things:
```
$ vim /etc/mkinitcpio.conf
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt lvm2 filesystems btrfs)
$ mkinitcpio -p linux
$ mkinitcpio -p linux-lts
```

$ pacman -S grub amd-ucode efibootmgr networkmanager network-manager-applet dialog wpa_supplicant mtools dosfstools reflector base-devel linux-headers bluez  bluez-utils cups hplip alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack bash-completion openssh rsync acpi acpi_call tlp sof_firmware acpid os-prober ntfs-3g nvidia nvida-utils nvidia-settings man

$ pacman -S grub efibootmgr dosfstools os-prober mtools

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

grub-mkconfig -o /boot/grub/grub.cfg

enable os-prober
vim /etc/default/grub 	# uncomment GRUB_DISABLE_OS_PROBER=false
grub-mkconfig -o /boot/grub/grub.cfg

systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable cups
systemctl enable sshd
systemctl enable tlp
systemctl enable reflector.timer
systemctl enable fstrim.timer
systemctl enable acpid


exit

umount -R /mnt

reboot

## after reboot 

nmuti 	# Activate connection

sudo grub-mkconfig -o /boot/grub/grub.cfg 	# it should detect windows boot manager as well
sudo pacman -S git

$ git clone https://aur.archlinux.org/yay-bin

$ cd yay-bin
$ makepkg -si

$ yay -S timeshift-bin timeshift-autosnap

$ sudo timeshift --list-devices

$ sudo timeshift --snapshot-devices

$ sudo timeshift --create --comments "FIrst backup" --tags D

# btrfs record snapstop automatically on system update/upgrade
$ grub-mkconfig -o /boot/grub/grub.cfg		# this will record timeshift snapshot as well

