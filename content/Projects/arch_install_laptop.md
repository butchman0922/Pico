/*
Title: Arch_on_Thinkpad
Description: ButchmanHome.net Projects
Author: Butch Wayman(butchman0922@gmail.com)
Date: 2015/01/31
Robots: noindex,nofollow
Template: index
*/

## Install Arch Linux on a Lenovo Thinkpad SL510

### Scheme:

  * gpt
  * luks on lvm
  * syslinux
  * auto login to tty
  * auto startx
  * auto start openbox WM


### Download Arch Linux:
  1. version
  
        archlinux-2014.12.01-dual.iso
        archlinux-2014.12.01-dual.iso.sig
  
  2. check PGP signature 
        
        pacman-key -v archlinux-2014.12.01-dual.iso.sig
  
  3. check md5sum
    
        md5sum archlinux-2014.12.01-dual.iso
   
  4. write to usbstick
    
        dd bs=4M if=/path/to/archlinux.iso of=/dev/sda && sync
    
  5. boot laptop from usbstick
  
### Partition and Format /dev/sda:
  
  1. setup gpt partition table
    
        gdisk /dev/sda
            o - write new gpt table
            n - create 200M boot partition
            n - create LVM partition using rest of drive - 232.7G
            w - write table and quit
    
  2. prepare boot dev
    * set "legacy_boot" attribute
    
            sgdisk /dev/sda --attributes=1:set:2
    
    * install to the MBR
      
            dd bs=440 conv=notrunc count=1 if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/sda

  3. format boot partition
      
        mkfs.ext2 /dev/sda1

  4. encrypt root partition
    
        cryptsetup luksFormat /dev/sda2
    
  5. open the encrypted container
    
        cryptsetup open --type luks /dev/sda2 lvm
    
  6. create a physical volume
    
        pvcreate /dev/mapper/lvm
    
  7. create volume group
    
        vgcreate LaptopHD /dev/mapper/lvm
    
  8. create logical volumes
    
        lvcreate -L 8G LaptopHD -n swapvol
        lvcreate -L 20G LaptopHD -n rootvol
        lvcreate -L 15G LaptopHD -n varvol
        lvcreate -l +100%FREE LaptopHD -n homevol
    
  9. format filesystems
    
        mkfs.ext4 /dev/mapper/LaptopHD-rootvol
        mkfs.ext4 /dev/mapper/LaptopHD-varvol
        mkfs.ext4 /dev/mapper/LaptopHD-homevol
        mkswap /dev/mapper/LaptopHD-swapvol
   
  10. mount filesystems start swap

        mount /dev/LaptopHD/rootvol /mnt
        mkdir /mnt/var
        mount /dev/LaptopHD/varvol /mnt/var
        mkdir /mnt/home
        mount /dev/LaptopHD/homevol /mnt/home
        mkdir /mnt/boot
        mount /dev/sda1 /mnt/boot
        swapon /dev/LaptopHD/swapvol
    
    
### Install base system:
 
  1.  connect to internet
    
        wifi-menu
  
  2. back up the existing mirrorlist
    
        cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

  3. uncomment mirrors for testing with rankmirrors
    
        nano /etc/pacman.d/mirrorlist.backup

  4. rank mirrors
    
        rankmirrors -n 5 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
    
  5. enable multilib
    
        nano /etc/pacman.conf

  6. install base packages
    
        pacstrap /mnt base
    

### Configure system:
    
  1.generate fstab file
    
        genfstab -U -p /mnt >> /mnt/etc/fstab
  
  2. change root
    
        arch-chroot /mnt
    
  3. set hostname
    
        echo hackbox > /etc/hostname

  4. set time zone
    
        ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime

  5. uncomment locales 
    
        nano /etc/locale.gen
    
  6. generate locales
    
        locale-gen
    
  7. set system locale preferences
    
        nano /etc/locale.conf
            LANG=en_US.UTF-8
            LC_COLLATE=C
  
  8. set console keymap and font
    
        loadkeys us
        setconsolefont lat2a-16
        nano /etc/vconsole.conf
            KEYMAP=us
            FONT=lat2a-16
      
### Install and Configure bootloader:

  1. install 
    
        pacman -S syslinux perl-passwd-md5 perl-digest-sha1 mtools gptfdisk dosfstools
    
  2. setup syslinux
    
        syslinux-install_update -i -a -m

  3. configure ramdisk
  
        nano /etc/mkinitcpio.conf
            HOOKS add lvm2 encrypt consolefont keymap
            MODULES add i915
    
  4. create new ramdisk   
    
        mkinitcpio -p linux

  5. set root password
    
        passwd
    
### Reboot:

  1. exit chroot environment
    
        exit
    
  2. unmount partitions
    
        umount -R /mnt
    
  3. reboot
  
### Setup network security:

  1. login as root

  2. create iptable chains
    
        iptables -N TCP
        iptables -N UDP
        iptables -P FORWARD DROP
        iptables -P OUTPUT ACCEPT
        iptables -P INPUT DROP
        iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
        iptables -A INPUT -p icmp --icmp-type 8 -m conntrack --ctstate NEW -j ACCEPT
        iptables -A INPUT -p udp -m conntrack --ctstate NEW -j UDP
        iptables -A INPUT -p tcp --syn -m conntrack --ctstate NEW -j TCP
        iptables -A INPUT -p udp -j REJECT --reject-with icmp-port-unreachable
        iptables -A INPUT -p tcp -j REJECT --reject-with tcp-rst
        iptables -A INPUT -j REJECT --reject-with icmp-proto-unreachable
        iptables -A TCP -p tcp --dport 22 -j ACCEPT
    
  3. save rules 
    
        iptables-save > /etc/iptables/iptables.rules
  
  4. start and enable iptables deamon   
    
        systemctl start iptables.service
        systemctl enable iptables.service
        systemctl status iptables.service

### Setup basic networking:

  1. plugin ethernet cable & usbwifi

  2. activate wired network device
        
        ip link set enp8s0 up
    
  3. get an address 
    
        dhcpcd enp8s0
      
  4. install netctl optional dependicies  
    
        pacman -S ifplugd wpa_supplicant wpa_actiond dialog
    
  5. set up wired netctl profiles
    
        cp /etc/netctl/examples/ethernet-dhcp /etc/netctl/home_lan
        nano /etc/netctl/home_lan
    
  6. take wired network device down 
    
        ip link set enp8s0 down
    
  7. set wireless netctl profiles
    
        wifi-menu -o wlp5s0
        netctl stop wlp5s0
        wifi-menu -o wlp0s29f7u2
        netctl stop wlp0s29f7u2
        nano /etc/netctl/home_usbwifi
            ExcludeAuto=yes
    
  8. configure network services
     
        systemctl enable netctl-ifplugd@enp8s0.service
        systemctl enable netctl-auto@wlp5s0.service
     
  9. reboot
  
### Setup autologin to virtual console:

  1. add packages
    
        pacman -Syu sudo zsh vim xf86-video-intel lib32-mesa-libgl xorg-init xorg-server xorg-server-utils xorg-apps  xf86-input-synaptics acpid
     
  2. enable gpm service
    
        systemctl enable gpm.service
    
  3. start network time service
    
        timedatectl set-ntp true 
  
  4. add user
    
        useradd -m -G wheel -s /bin/zsh butchman
        chfn Butch Wayman
        passwd butchman
    
  5. setup sudo
    
        EDITOR=nano visudo
    
  6. logout
  
  7. login as butchman
  
  8. set autologin to tty
    
        sudo nano /etc/systemd/system/getty@tty1.service.d/autologin.conf
            [Service]
            ExecStart=-/usr/bin/agetty --autologin butchman --noclear %I 38400 linux
      
  9. startx at login
    
        sudo nano .zprofile
            [[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && exec startx
      
  10. setup openbox
      
        cp /etc/skel/.xinitrc ~
        nano .xinitrc
            exec openbox-session
        
### TODO: 

  1. configure silent boot and/or 
  
        https://wiki.archlinux.org/index.php/Silent_boot
      
  2. setup acpi
  
    
  
    
  
  
    
    





    
