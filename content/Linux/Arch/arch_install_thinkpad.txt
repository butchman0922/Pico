#Install arch linux to Thinkpad sl510

##Scheme:
  -gpt
  -luks on lvm
  -syslinux
  -auto login to tty
  -auto startx
  -auto start openbox WM


##Download Arch Linux:
  version 
    archlinux-2014.12.01-dual.iso
    archlinux-2014.12.01-dual.iso.sig
  
  check PGP signature 
    pacman-key -v archlinux-2014.12.01-dual.iso.sig
  
  check md5sum
    md5sum archlinux-2014.12.01-dual.iso
   
  write to usbstick
    dd bs=4M if=/path/to/archlinux.iso of=/dev/sda && sync
    
  boot laptop from usbstick
  
##Partition and Format /dev/sda:
  setup gpt partition table:
    gdisk /dev/sda
      o - write new gpt table
      n - create 200M boot partition
      n - create LVM partition using rest of drive - 232.7G
      w - write table and quit
    
  prepare boot dev
    set "legacy_boot" attribute
      sgdisk /dev/sda --attributes=1:set:2
    install the MBR
      dd bs=440 conv=notrunc count=1 if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/sda

  format boot partition
      mkfs.ext2 /dev/sda1

  encrypt root partition
    cryptsetup luksFormat /dev/sda2
    
  open the encrypted container
    cryptsetup open --type luks /dev/sda2 lvm
    
  create a physical volume
    pvcreate /dev/mapper/lvm
    
  create volume group
    vgcreate LaptopHD /dev/mapper/lvm
    
  create logical volumes
    lvcreate -L 8G LaptopHD -n swapvol
    lvcreate -L 20G LaptopHD -n rootvol
    lvcreate -L 15G LaptopHD -n varvol
    lvcreate -l +100%FREE LaptopHD -n homevol
    
  format filesystems
    mkfs.ext4 /dev/mapper/LaptopHD-rootvol
    mkfs.ext4 /dev/mapper/LaptopHD-varvol
    mkfs.ext4 /dev/mapper/LaptopHD-homevol
    mkswap /dev/mapper/LaptopHD-swapvol
   
  mount filesystems start swap
    mount /dev/LaptopHD/rootvol /mnt
    mkdir /mnt/var
    mount /dev/LaptopHD/varvol /mnt/var
    mkdir /mnt/home
    mount /dev/LaptopHD/homevol /mnt/home
    mkdir /mnt/boot
    mount /dev/sda1 /mnt/boot
    swapon /dev/LaptopHD/swapvol
    
    
 ##Install base system:
 
  connect to internet
    wifi-menu
  
  back up the existing mirrorlist
    cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

  uncomment mirrors for testing with rankmirrors
    nano /etc/pacman.d/mirrorlist.backup

  rank mirrors
    rankmirrors -n 5 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
    
  enable multilib
    nano /etc/pacman.conf

  install base packages
    pacstrap /mnt base
    

##Configure syatem:
    
  generate fstab file
    genfstab -U -p /mnt >> /mnt/etc/fstab
  
  change root
    arch-chroot /mnt
    
  set hostname
    echo hackbox > /etc/hostname

  set time zone
    ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime

  uncomment locales 
    nano /etc/locale.gen
    
  generate locales
    locale-gen
    
  set system locale preferences
    nano /etc/locale.conf
      LANG=en_US.UTF-8
      LC_COLLATE=C
  
  set console keymap and font
    loadkeys us
    setconsolefont lat2a-16
    nano /etc/vconsole.conf
      KEYMAP=us
      FONT=lat2a-16
      
##Install and Configure bootloader:

  install 
    pacman -S syslinux perl-passwd-md5 perl-digest-sha1 mtools gptfdisk dosfstools
    
  setup syslinux
    syslinux-install_update -i -a -m

  configure ramdisk
    nano /etc/mkinitcpio.conf
      HOOKS add lvm2 encrypt consolefont keymap
      MODULES add i915
    
    
  create new ramdisk   
    mkinitcpio -p linux

  set root password
    passwd
    
##Reboot:

  exit chroot environment
    exit
    
  unmount partitions
    umount -R /mnt
    
  reboot
  
##Setup network security:

  login as root

  create iptable chains
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
    
  save rules 
    iptables-save > /etc/iptables/iptables.rules
  
  start and enable iptables deamon   
    systemctl start iptables.service
    systemctl enable iptables.service
    systemctl status iptables.service

##Setup basic networking:

  plugin ethernet cable & usbwifi

  activate wired network device
    ip link set enp8s0 up
    
  get an address 
    dhcpcd enp8s0
      
  install netctl optional dependicies  
    pacman -S ifplugd wpa_supplicant wpa_actiond dialog
    
  set up wired netctl profiles
    cp /etc/netctl/examples/ethernet-dhcp /etc/netctl/home_lan
    nano /etc/netctl/home_lan
    
  take wired network device down 
    ip link set enp8s0 down
    
  set wireless netctl profiles
    wifi-menu -o wlp5s0
    netctl stop wlp5s0
    wifi-menu -o wlp0s29f7u2
    netctl stop wlp0s29f7u2
    nano /etc/netctl/home_usbwifi
      ExcludeAuto=yes
    
  configure network services
     systemctl enable netctl-ifplugd@enp8s0.service
     systemctl enable netctl-auto@wlp5s0.service
     
  reboot
  
##Setup autologin to virtual console:

  add packages
    pacman -Syu sudo zsh vim xf86-video-intel lib32-mesa-libgl\
     xorg-init xorg-server xorg-server-utils xorg-apps  xf86-input-synaptics\
     acpid
     
  enable gpm service
    systemctl enable gpm.service
    
  start network time service
    timedatectl set-ntp true 
  
  add user
    useradd -m -G wheel -s /bin/zsh butchman
    chfn Butch Wayman
    passwd butchman
    
  setup sudo
    EDITOR=nano visudo
    
  logout
  
  login as butchman
  
  set autologin to tty
    sudo nano /etc/systemd/system/getty@tty1.service.d/autologin.conf
      [Service]
      ExecStart=
      ExecStart=-/usr/bin/agetty --autologin butchman --noclear %I 38400 linux
      
   startx at login
    sudo nano .zprofile
      [[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && exec startx
      
   setup openbox
      cp /etc/skel/.xinitrc ~
      nano .xinitrc
        exec openbox-session
        
   setup acpi
    
        
        
##TODO: 
  1. configure silent boot and/or 
      https://wiki.archlinux.org/index.php/Silent_boot
      

  
    
  
    
  
  
    
    





    
