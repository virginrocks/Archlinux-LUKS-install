# Install Archlinux on LVM LUKS2 encrypted disk

## Features

1 Iso
Download iso and prepare arch iso bootable usb.

2 LUKS LVM
Prepare the disk, set LUKS2 encryption, create physical volume, volume group and logical volumes.

3 Install arch
Pacstarp on /mnt, generate /etc/fstab, arch-chroot /mnt and setup the clock, users, network...

4 Bootctl and mkinitcpio
Setup mkinitcpio.conf, install bootctl, create arch.conf and loader.conf.

5 Boot from installed system
Install usefull utilities.

6 LUKS Keyfile
Generate keyfile, add to LUKS2, setup cmdline, linux.preset, crypttab.initramfs.

# 1 Iso

## Creating usb bootable installer

### Downloading iso

### Cpying iso on usb device

    sudo dd if=path/to/file.iso of=/dev/usb_drive status=progress

## Install process

### Boot on usb: 

make sure scure boot is disabled on BIOS
select usb device for boot

## Welcome on ArchLinux!

### if your keyboard is azerty, just load the right configuration by

    loadkeys fr

### setup wifi if needed

    iwctl station wlan0 get-networks

### you should see the SSID of your wifi router

    iwctl station wlan0 connect SSID

### Update sources:

    pacman -Sy

# 2 LUKS LVM disk architecture

## Prepare the disk

### List your partitions

    lsblk            # list drive

### Erase and create partitions: parted, fdisk, cfdisk...choose a disk manager 

    parted /dev/sdX  # enter parted prompt
    p            
    mklabel gpt  
    mkpart ESP fat32 1MiB 513MiB
    set 1 esp on
    mkpart primary 513MiB 100%
    p
    q            

### Clean up

    dd if=/dev/urandom of=/dev/sdX1 bs=1M status=progress 
    dd if=/dev/urandom of=/dev/sdX2 bs=1M status=progress 

### Boot partition

    mkfs.vfat -F32 /dev/sdX1

### Encryption: create the LUKS container and open the container

    cryptsetup -v luksFormat /dev/sdX2 
    cryptsetup luksOpen /dev/sdX2 lvm

### Create physical volume ex: named lvm

    pvcreate /dev/mapper/lvm

### Create a volume group ex: named vg

    vgcreate vg /dev/mapper/lvm

### Create logical volumes
    
    lvcreate -L 60G vg -n root        
    lvcreate -L 8G vg -n swap       
    lvcreate -l 100%FREE vg -n home        
    lvreduce -L -256M vg/home   

### Check
    
    lsblk -fp        

### Format file system

    mkfs.ext4 /dev/vg/root    
    mkfs.ext4 /dev/vg/home    
    mkswap /dev/vg/swap    

# 3 Install Archlinux

### Mount file system

    mount /dev/vg/root /mnt
    mount --mkdir -o uid=0,gid=0,fmask=0077,dmask=0077 /dev/sda1 /mnt/boot
    mount --mkdir /dev/vg/home /mnt/home
    swapon /dev/vg/swap

### Install minimum packages

    pacstrap -K /mnt base linux-lts linux-firmware linux-headers intel-ucode sudo vim

### Generate fstab

    genfstab -U /mnt >> /etc/fstab    

### Archlinux chroot

    arch-chroot /mnt

### Time zone

    ln -sf /usr/share/zoneinfo/Region/City /etc/localetime

### Enable NTP

    timedatectl set-ntp true

### Generate /etc/adjtime  

    hwclock --systohc    

### Genreate locale and uncomment the line matching your language in /etc/locale.gen

    locale-gen

### Create vconsole.conf

    touch /etc/vconsole.conf && echo -e "FONT=lat1-16 \nKEYMAP=fr-latin9" > /etc/vconsole.conf         

### Create /etc/hostname

    touch /etc/hostname && echo "yourHostName" > /etc/hostname    

### Modify /etc/hosts

    echo "127.0.0.1       yourHostName.localdomain localdomain" >> /etc/hosts 

### Set password for root and non-root user
    
    passwd    
    useradd -m -G wheel,storage,audio,video -s /bin/bash yourUserName
    passwd yourUserName

### Uncomment #%wheel ALL=(ALL) ALL in /etc/sudoers to allow non-root user to run sudo 
 
    %wheel ALL=(ALL) ALL

### Network

    pacman -S dhcpcd iw iwd
    systemctl enable iw.service
    systemctl enable dhcpcd.service

# 4 Bootctl and mkinitcpio.conf

### Bootctl
 
    bootctl install

### Set mkinitcpio for LUKS encrypt: add sd-encrypt lvm2 before block
 
    MODULES=( ext4 dm-mod dm-crypt )
    HOOKS=(...sd-encrypt lvm2 block ...)

### Install lvm2
 
    pacman -S lvm2

### Get /dev/sdX2 UUID
 
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

# 5 Reboot without installer 

###### Follow https://github.com/silentz/arch-linux-install-guide for usefull utilities

# 6 LUKS keyfile, UKIFY, cmdline and crypttab

## Skip the LUKS passphrase:

### Create a key and store it in /etc/crypsetup.d
    
    dd bs=512 count=4 if=/dev/urandom iflag=fullblock | install -m 600 /dev/sdX2 /etc/cryptsetup.d/root.key 

### Check the slots already used for the encryption keys 
 
    cryptsetup luksDump/dev/sdX2

### Associate this key to your LUKS setup
 
    cryptsetup luksAddKey /dev/sdX2 /etc/cryptsetup-key.d/root.key

### Check the slots again
 
    cryptsetup luksDump/dev/sdX2

### Update mkinitcpio.conf
    
    FILES=( /etc/cryptsetup-keys.d/root.key )
    HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap consolefont sd-vconsole sd-encrypt lvm2 block filesystems fsck)

### Install ukify
 
    pacman -S ukify-tools sbctl

    vim /etc/kernel/uki.conf

### Create /etc/kernel/uki.conf:

    [UKI]
    OSRelease=@/etc/os-release
    PCRBanks=sha256

    [PCRSignature:initrd]
    Phases=enter-initrd
    PCRPrivateKey=/etc/kernel/pcr-initrd.key.pem
    PCRPublicKey=/etc/kernel/pcr-initrd.pub.pem

### Generate the key

    ukify genkey --config=/etc/kernel/uki.conf    

### Create /etc/kernel/cmdline  
###### cmdline is kept simple to avoid errors with absolute path
 
    root=/dev/vg/root rw rd.system.gpt_auto=no quiet splash

### Setup /etc/mkinitcpio.d/linux.preset: example
 
    # mkinitcpio preset file for the 'linux' package

    #ALL_config="/etc/mkinitcpio.conf"
    #ALL_kver="/boot/vmlinuz-linux-lts"
    ALL_kerneldest="/boot/vmlinuz-linux-lts"

    #PRESETS=('default')
    PRESETS=('default' 'fallback')

    #default_config="/etc/mkinitcpio.conf"
    #default_image="/boot/initramfs-linux-lts.img"
    default_uki="/boot/EFI/Linux/arch-linux.efi"
    default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

    #fallback_config="/etc/mkinitcpio.conf"
    #fallback_image="/boot/initramfs-linux-fallback.img"
    fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
    fallback_options="-S autodetect"

### Create and setup /etc/crypttab.initramfs so you will not be blocked at boot
 
    lvm      UUID=xxxxxxxxxxxx   /etc/cryptsetup-key.d/root/key  luks

### Genrate secure boot keys

    sbctl create-keys

### Update and reboot

    mkinitcpio -P
    bootctl update
    reboot


	


