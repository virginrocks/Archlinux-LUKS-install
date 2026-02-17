# :herb: LUKS encrypted LVM Arch Linux Install guide 

## Presentation

This page shows one of the multiple possibilities for installing Archlinux on LUKS2 encrypted LVM disk.

This example shows a manual install process, the simple way is indeed to use native's arch installation script.

⚠️   This installation uses an *_hybrid_* *EFI and boot* setup using systemd-boot, where the fat32 ESP is mounted at boot, mixing UEFI files with Linux kernel files. 

:bulb: This configuration does not allowed /boot encryption, and there is a risk of _conflicts_ in dual boot cases.

## Features

1. Iso

:cd: Download iso and prepare arch iso bootable usb.

2. LUKS LVM

:dvd: Prepare the disk, set LUKS2 encryption, create physical volume, volume group and logical volumes.

3. Install arch

Pacstarp on /mnt, generate /etc/fstab, arch-chroot /mnt and setup the clock, users, network...

4. Bootctl and mkinitcpio

Setup mkinitcpio.conf, install bootctl, create arch.conf and loader.conf.

5. Boot from installed system

Install usefull utilities.

6. LUKS Keyfile

Generate keyfile, add to LUKS2, setup cmdline, linux.preset, crypttab.initramfs.

# 1. Iso

## Creating usb bootable installer

### Downloading iso

### Copying iso on usb device

    sudo dd if=path/to/file.iso of=/dev/usb_drive status=progress

## Install process

### Boot on usb:

🔔 Make sure sure Secure Boot is disabled on BIOS

Select usb device for boot

## Welcome on ArchLinux!

### To change your keyboard layout

    loadkeys fr

### Setup wifi or plug ethernet

The arch installer provides a large set of tools, including the wifi network tool iwctl

    iwctl station wlan0 get-networks

### You should see the SSID of your wifi router

    iwctl station wlan0 connect YOUR_SSID

### Update sources:

    pacman -Sy

# 2. LUKS LVM disk architecture

## :dvd: Prepare the disk

:bulb: Assuming that you start an installation with empty or erasable disk space

If you wish create a dual boot architecture, do not follow this steps

### List your partitions

    lsblk            # list drive

### Erase and create partitions: parted, fdisk, cfdisk...choose a disk manager 

Let's use parted, but choice is yours

Enter parted

    parted /dev/sdX  

Printing detailed gpt infos including labels

    p                   

Install GUID Partition Table (GPT)

The gpt stores human readable labels in the partition entry array at the disk start and mirrored at the end for redundancy 

This stands for the modern standard for disk partitionning on UEFI systems and large drives (old system is MBR)

    mklabel gpt     

Let's create first partition for arch boot, dev/sdX1, in which will be installed both kernel (vmliuz, initramfs) and UEFI (grub or systemd boot) files.

:vs: Another solution would be to create two partitions: one in ext4 or xfs to store the kernel files, and one in fat32 to store UEFI files in order to keep them separate from each other 

| Aspect             | Single /boot           | Dual /boot + /boot/efi                |
| ------------------ | ---------------------- | ------------------------------------- |
| UEFI Compliance    | Partial (FAT32 only)   | Full (dedicated ESP)                  |
| Encryption Support | Limited (no LUKS root) | Strong (LUKS root + optional /boot)   |
| Size/Complexity    | Smaller, simpler       | More partitions, but standard         |
| Use Case           | BIOS or minimal UEFI   | Modern UEFI + encryption              |

    mkpart esp fat32 1MiB 1025MiB 

Set boot flag on boot partition /dev/sdX1

    set 1 esp on                 

Create main partition, which will be later encrypted and sliced into logical volumes (lv) within both physical and group volumes (pv & vg)

    mkpart primary 1025MiB 100%   

Print and check

    p                            

Save and exit parted utility

    q                            

### Clean up

A good practice is to erase the disk by writing random data sequences with dd

Note: Can take a while, replace urandom with zero to make it faster if your time is limited

    dd if=/dev/urandom of=/dev/sdX1 bs=1M status=progress 
    dd if=/dev/urandom of=/dev/sdX2 bs=1M status=progress 

### Create fat32 file system on esp

Fat32 can store both UEFI and Kernel files, the opposite isn't true (we can't store UEFI files in Linux file system)

    mkfs.vfat -F32 /dev/sdX1

### Create a LUKS2 encrypted container and open it under the name of lvm

First, the main partition is going to be fully encrypted with LUKS2

The passphrase you will give will be stored by LUKS (cryptsetup luksDump /dev/sdx2 to print LUKS header informations)

:bulb: Avoid to loose your passphrase before you create a keyfile and a recovery key

    cryptsetup -v luksFormat /dev/sdX2   
    cryptsetup luksOpen /dev/sdX2 lvm   

### Logical Volume Management (LVM)

Here is the LVM part, wich allows great flexibility as it can be easily resized without data loss and allows disks to be snapshooted

| Aspect       | Static Partitions                     | LVM                                 |
| ------------ | ------------------------------------- | ----------------------------------- |
| I/O Overhead | Minimal, direct access  | Slight due to layers |
| Resize Speed | Offline, complex              | Online, straightforward      |
| Multi-Disk   | Manual spanning needed                | Native pooling             |

### Create physical volume named lvm

    pvcreate /dev/mapper/lvm

### Create a volume group named vg

    vgcreate vg /dev/mapper/lvm

### Create logical volumes root, swap and home
    
    lvcreate -L 60G vg -n root        
    lvcreate -L 8G vg -n swap       
    lvcreate -l 100%FREE vg -n home        
    lvreduce -L -256M vg/home   

### Check volumes
    
    lsblk -fp        

### Format filesystems (eg. ext4 or other like xfs)

    mkfs.ext4 /dev/vg/root    
    mkfs.ext4 /dev/vg/home    
    mkswap /dev/vg/swap    

# 3. Install Archlinux

### Mount file system

    mount /dev/vg/root /mnt # eg. /dev/mapper/vg-root
    mount --mkdir -o uid=0,gid=0,fmask=0077,dmask=0077 /dev/sda1 /mnt/boot
    mount --mkdir /dev/vg/home /mnt/home
    swapon /dev/vg/swap

### Install minimum packages

    pacstrap -K /mnt base linux-lts linux-firmware linux-headers intel-ucode sudo vim

### Generate fstab

    genfstab -U /mnt >> /mnt/etc/fstab    

### Enter the chroot environment

    arch-chroot /mnt

### :date: Time zone setup

    ln -sf /usr/share/zoneinfo/Region/City /etc/localetime

### Enable NTP

    timedatectl set-ntp true

### Generate /etc/adjtime  

    hwclock --systohc    

### Uncomment the line matching your language in /etc/locale.gen

    fr_CH.UTF-8 UTF-8

##  Generate locale 
    
    locale-gen

### Create vconsole.conf

    touch /etc/vconsole.conf && echo -e "FONT=lat1-16 \nKEYMAP=fr-latin9" > /etc/vconsole.conf         

### Create /etc/hostname

    touch /etc/hostname && echo "yourHostName" > /etc/hostname    

### Modify /etc/hosts

    echo "127.0.0.1       yourHostName.localdomain localdomain" >> /etc/hosts 

### Set password for root and non-root user
    
    passwd    
    useradd -m -G wheel,storage,audio,video -s /bin/bash yourUserName # -m creates user's folder
    passwd yourUserName

### Uncomment #%wheel ALL=(ALL) ALL in /etc/sudoers to allow non-root user to run sudo 
 
    %wheel ALL=(ALL) ALL

### Network setup 

    pacman -S dhcpcd iw iwd
    systemctl enable iw.service
    systemctl enable dhcpcd.service

# 4. Bootctl and mkinitcpio.conf

### Bootctl setup (You can choose grub also)  

    bootctl install 

### Take a look at bootctl

    bootctl status
    bootctl list

### Set mkinitcpio 

Edit /etc/mkinitcpio.conf, add sd-encrypt lvm2 before block in HOOKS, and answer MODULES 
 
    MODULES=( ext4 dm-mod dm-crypt )
    HOOKS=(...sd-encrypt lvm2 block ...)

### Install lvm2
 
    pacman -S lvm2

### Get /dev/sdX2 UUID

Get LUKS UUID to allow the decryption process at boot

At this point, you will have to give the disk passphrase at reboot

Later in this document is described the process for enhancing the boot sequence by setting a keyfile, which allows booting without passphrae
 
    blkid -s UUID -o value /dev/sdX2  # LUKS UUID

### Set /boot/loader/entries/arch.conf
 
    title	Arch Linux
    linux	/vmlinuz-linux-lts
    initrd	/initramfs-linux-lts.img
    initrd	/intel-ucode.img

    options systemd.unit=multi-user.target rd.luks.name=$(blkid -s UUID -o value /dev/sdX2)=lvm root=/dev/vg/root rw
 
### Set /boot/loader/loader.conf
 
    default arch.conf
    timeout 3
    console-mode max
    editor no

### Reload mkinitcpio

    mkinitcpio -P
    bootctl update

### Cleanly exit and reboot

#### Exit from chroot

    exit

#### Unmount file system from /mnt

    umount -R /mnt

#### Deactivate logical volumes in volume group

    vgchange -an

#### Close LUKS container

    crytsetup luksClose lvm

    reboot

# 5. Reboot on Arch and install packages ✅

##### Follow https://github.com/silentz/arch-linux-install-guide for usefull utilities

# 6. LUKS keyfile, UKIFY, cmdline and crypttab

As earlyer mentioned, the LUKS passphrase prompt can be skipped by creating a dedicated keyfile 

### Create the keys direcory

    sudo mkdir -p /etc/cryptsetup-keys.d

### :key: Create a key and store it in /etc/crypsetup.d

The following command does two things: generate a 2Ko key and puts it in /etc/cryptsetu-keys.d with permissions -r-x------.
    
    sudo dd bs=512 count=4 if=/dev/urandom iflag=fullblock | sudo install -m 600 /dev/stdin /etc/cryptsetup-keys.d/root.key 

### Check the slots already used for the encryption keys 
 
    sudo cryptsetup luksDump/dev/sdX2

### :lock: Associate this key to your LUKS setup
 
    sudo cryptsetup luksAddKey /dev/sdX2 /etc/cryptsetup-keys.d/root.key

### Check the slots again
 
    sudo cryptsetup luksDump/dev/sdX2

### Update mkinitcpio.conf
    
    FILES=( /etc/cryptsetup-keys.d/root.key )
    HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap consolefont sd-vconsole sd-encrypt lvm2 block filesystems fsck)

### Install ukify

##### Clone github's AUR repository 

    git clone https://aur.archlinux.org/dracut-ukify.git

##### Go in the folder

    cd dracut-ukify

##### Build package 

    makepkg -si

### Create /etc/kernel/uki.conf:

    sudo vim /etc/kernel/uki.conf

### Add the folowing to uki.conf:

    [UKI]
    OSRelease=@/etc/os-release
    PCRBanks=sha256

    [PCRSignature:initrd]
    Phases=enter-initrd
    PCRPrivateKey=/etc/kernel/pcr-initrd.key.pem
    PCRPublicKey=/etc/kernel/pcr-initrd.pub.pem

### Generate the key

    sudo ukify genkey --config=/etc/kernel/uki.conf    

### Install sbctl
 
    sudo pacman -S sbctl

### Create /etc/kernel/cmdline  

    sudo vim /etc/kernel/cmdline

### cmdline is kept simple to avoid errors with absolute path
 
    root=/dev/vg/root rw rd.system.gpt_auto=no quiet splash

### Setup /etc/mkinitcpio.d/linux.preset: example
 
    # mkinitcpio preset file for the 'linux' package

    #ALL_config="/etc/mkinitcpio.conf"
    ALL_kver="/boot/vmlinuz-linux-lts"
    ALL_kerneldest="/boot/vmlinuz-linux-lts"

    PRESETS=('default')
    #PRESETS=('default' 'fallback')

    #default_config="/etc/mkinitcpio.conf"
    #default_image="/boot/initramfs-linux-lts.img"
    default_uki="/boot/EFI/Linux/arch-linux.efi"
    default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

    #fallback_config="/etc/mkinitcpio.conf"
    #fallback_image="/boot/initramfs-linux-fallback.img"
    #fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
    #fallback_options="-S autodetect"

### Create and setup /etc/crypttab.initramfs so you will not be blocked at boot
 
    lvm      UUID=$(blkid -s UUID -o value /dev/sdX2)   /etc/cryptsetup-keys.d/root.key  luks

### Backup LUKS header

    sudo cryptsetup luksHeaderBackup /dev/sdX2 --header-backup-file ~/sdX2-header-$(date +F%).img

### Create a recovery key for your LUKS

    sudo systemd-cryptenroll /dev/sdX2 --recovery-key
        🔐 Please enter current passphrase for disk /dev/sdX2: ••••••••••
        A secret recovery key has been generated for this volume:

             🔐 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

        Please save this secret recovery key at a secure location. It may be used to
        regain access to the volume if the other configured access credentials have
        been lost or forgotten. The recovery key may be entered in place of a password
        whenever authentication is requested.
        New recovery key enrolled as key slot x.

### Check the recovery key (Only possible if /dev/sdX2 is not mounted)

    sudo systemd-cryptsetup attach test /dev/sdX2
    🔐 Please enter passphrase or recovery key for disk primary (test): •••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••••
    Set cipher aes, mode xts-plain64, key size 512 bits for device /dev/sdX2.

### Edit loader.conf

    sudo vim /boot/loader/loader.conf

### Comment all to hide unecessary  boot console 

If there is only Arch to boot with bootctl, there's no menu.

In the case of dual boot, this file becomes important to display boot menu.

    #default arch.conf
    #timeout 3
    #console-mode max
    #editor no

### Update and reboot

    sudo mkinitcpio -P
    sudo bootctl update
    reboot

### Genrate secure boot keys

    sudo sbctl create-keys

### Reboot to Setup Mode

To enroll sbctl keys, BIOS secure mode must be set to Setup Mode

Refer to your machine's bios manual.

### Enroll secure bootkeys

The following option -m preserve vendor keys from being enrolled.  

    sudo sbctl enroll-keys -m

### Sign boot files with sbctl

    sudo sbctl sign-all
    sudo sbctl verify

### Update and reboot

    sudo mkinitcpio -P
    sudo bootctl update
    reboot

## ⏳ Next: Dual boot Arch Linux / Rocky Linux setup




	


