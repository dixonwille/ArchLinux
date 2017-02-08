# Arch Linux

This is the way that I settup my arch linux machine.

## Helpful Links

* [Partitioning](https://wiki.archlinux.org/index.php/partitioning)
* [Useful Guid](https://gist.github.com/mattiaslundberg/8620837)
* [CBelk Guid](https://github.com/cbelk/arch-install/blob/master/setup)

## Installation

Run through of everything I do to install arch linux onto a hard drive. The first part requries you to have an [Arch Live USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media).

### Partitions

Make sure that we are booted into the drive via EFI.

* `ls /sys/firmware/efi/efivars` this should exist and be populated

Use `lsblk` to determine which drive to use.

Run `parted /dev/sdX`. Turn the drive into a guid partition table using `mklabel gpt`. In parted we need to create a few partions:

* `mkpart efi fat32 1MiB 513MiB`    //EFI boot
* `set 1 boot on`                   //Make bootable
* `mkpart primary ext2 513MiB 1GiB` //Boot partition
* `set 2 boot on`                   //Make bootable
* `mkpart primary ext4 1GiB 100%`   //Everyting else
* `set 3 lvm on`                    //Use lvm for this partition

Now we need to format them.

* `mkfs.vfat -F32 /dev/sdX1`    //Format the EFI correctly
* `mkfs.ext2 /dev/sdX2`         //Format the boot correctly

Setup encryption on the rest of the drive.

* `cryptsetup -c aes-xts-plain64 -y --use-random luksFormat /dev/sdX3`  //sets the encrytion for everything else
* `cryptsetup luksOpen /dev/sdX3 lvm`                                  //opens it for use

Create the logical volumes inside the encrypted partition.

* `pvcreate /dev/mapper/lvm`                //Prepare logical volumes
* `vgcreate vg0 /dev/mapper/lvm`            //Create logical volume group
* `lvcreate --size 8G vg0 --name swap`      //Create swap logical volume (Half of RAM is my rule of thumb)
* Create additional volumes if wanted (home, var, etc.)
* `lvcreate -l +100%FREE vg0 --name root`   //Create the root with the rest of the space (usually your home takes up most of the space)

Format your logical drives.

* `mkfs.ext4 /dev/mapper/vg0-root`
* `mkswap /dev/mapper/vg0-swap`

Mount everything together. (Don't forget to mount your other partitions in the appropriate areas)

* `mount /dev/mapper/vg0-root /mnt` //Root needs to be mounted first
* `swapon /dev/mapper/vg0-swap`     //Turn the swap on (allows genfstab to include in fstab)
* `mkdir /mnt/boot`                 //Create the boot directory
* `mount /dev/sdX2 /mnt/boot`       //Mount the boot partition
* `mkdir /mnt/boot/efi`             //Make the efi directory
* `mount /dev/sdX1 /mnt/boot/efi`   //Mount the efi partition

### Install Arch

This command install arch onto `/mnt` with a few dependencies.

`pacstrap /mnt base base-devel linux-headers grub-efi-x86_64 zsh vim git efibootmgr dialog`

If you are using an intel cpu include `intel-ucode`.

Include `wpa_suppicant` if you are using a laptop with wifi capabilities.

If you want long term support kernal as well include `linux-lts` and `linux-lts-headers`. This is useful for a backup kernal incase your main kernal is bricked for any reason.

### Configure

Generate fstab with `genfstab -pU /mnt >> /mnt/etc/fstab`. This will mount everything together on boot like we did above.

The next few lines is to allow the mounted partitions see the efi variables for a clean grub configuration (can be removed on remount)

* `mkdir /mnt/hostrun/`
* `mount --bind /run /mnt/hostrun`
* `arch-chroot /mnt /bin/bash`           //This will drop you into your newly installed arch machine. 
* `mkdir /run/lvm`
* `mount --bind /hostrun/lvm /run/lvm`

Change the computers locales by editing `/etc/locale.gen` and uncomment the appropriate locale. Then run `locale-gen`. You will also need to create `/etc/locale.conf` and add `LANG=en_US.UTF-8` (replace with the approprite locale).

To setup your system clock run `ln -sf /usr/share/zoneinfo/ZONE/SUBZONE /etc/localtime` replacing ZONE and SUBZONE with the appropriate values. Then set you machine up to store the time as UTC using `hwclock --systohc --utc`.

Change the host name of the new machine `echo HOSTNAME > /etc/hostname`.

Create entries in `/etc/hosts` that point to localhost but use the host name you supplied above.

Change the root password using `passwd`.

Few things need to be edited in `/etc/mkinitcpio.conf`:

* Add `ext4` to `MODULES`
* Add `keymap`, `encrypt` and `lvm2` to `HOOKS` before filesystems

Then regenerate initrd images using `mkinitcpio -p linux` and `mkinitcpio -p linux-lts` (only use the latter if you installed lts as well).

For grub the following file should be edited `/ect/default/grub` with this change to `GRUB_CMDLINE_LINUX`:

* `cryptdevice=/dev/sdd3:lvm`

Now install grub with `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux`. Also to apply the configurate we need to run `grub-mkconfig -o /boot/grub/grub.cfg` (This is where mounting the hostrun comes in to play).

We have successfully (hopefully) installed arch on the machine. We need to exit, unmount, and reboot the system (don't forget to remove the live usb).

* `exit`
* `umount -R /mnt`
* `swapoff -a`
* `poweroff`

## PostInstall

On reboot you will be asked to supply the password to unlock the encrypted drive. You will also be booted into the lts kernal if you installed it (I change my grub configuration to default to the newest version).

There are a few more things we should do before we start configuring it for personal use. First login as root (as that is the only user available).

Create a user for yourself and add them to the wheel and users group. I am also changing the default shell to be zsh. Also create a password for your new user.

* `useradd -m -G wheel,users -s /bin/zsh username`
* `passwd username`

Run `visudo` and uncomment/add the line that reads `%wheel ALL=(ALL) ALL` to allow wheel group members sudo access (which you are).

Reboot the system and you should be able to login as your new user.

## Next Steps

Install all the fun and intresting packages available to you. I would suggest installing `xorg-server` and your appropriate drivers next if you want a Desktop Envrionment.

## Pakcages

These are packages that I usually install and configure after a fresh install to get everything up and functioning for me.

Packages are in order as I would install them if installing them separatly from others. Some of these packages to depend on others to be installed.

* connman
* ranger
* go
* go-tools
* alsa-util
* pulseaudio
* pulseaudio-alsa
* transmission-cli
* transmission-remote-cli
* keybase
* openssh

I ran these together to make sure they were tied to eachother.

* xorg-server
* xorg-server-utils
* xorg-apps
* nvidia
* nvidia-lts
* nvidia-libgl
* lib32-nvidia-libgl

This if for my desktop environment

* openbox
* feh
* autocutsel
* xcompmgr
* urxvt
* tint2
* libnotify
* firefox
* pavucontrol
* VSCode (AUR)
* vlc

## Configuration Helpful

* [GPG2](https://wiki.archlinux.org/index.php/GnuPG)
    * useful for setting up ssh

## TODO

* Add Dot files and other configuration files to this repository to help ease the process of configuring.