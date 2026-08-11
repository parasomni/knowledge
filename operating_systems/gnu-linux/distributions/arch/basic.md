# Basic minimal Arch Linux installation

## Table of contents

- [1. Create boot disk](#1-create-boot-disk)
- [2. WIFI Setup](#2-wifi-setup)
- [3. Prepare disks](#3-prepare-disks)
  - [3.1 Partitioning](#31-partitioning)
- [4. Formating](#4-formating)
  - [4.1 Create LUKS encrypted container](#41-create-luks-encrypted-container)
- [5. Mounting](#5-mounting)
- [6. Install minimal base system](#6-install-minimal-base-system)
- [7. Generate fstab](#7-generate-fstab)
- [8. Enter new system](#8-enter-new-system)
- [9. Set timezone](#9-set-timezone)
- [10. Configure locale](#10-configure-locale)
- [11. Set hostname](#11-set-hostname)
- [12. Add local user](#12-add-local-user)
- [13. Enable sudo](#13-enable-sudo)
- [14. Install bootloader](#14-install-bootloader)
  - [14.1 Configure mkinitcpio for LUKS](#141-configure-mkinitcpio-for-luks)
  - [14.2 UUID of encrypted partition](#142-uuid-of-encrypted-partition)
  - [14.3 Installation](#143-installation)
  - [14.4 Add keyboard](#144-add-keyboard)
  - [Reboot](#reboot)
- [Hardening setup](#hardening-setup)
  - [Install required security packages](#install-required-security-packages)
  - [Kernel hardening](#kernel-hardening)
  - [Secure SSH](#secure-ssh)
  - [Password policy](#password-policy)
  - [Account lockout - brute force protection](#account-lockout---brute-force-protection)
  - [File permissions](#file-permissions)
  - [Disable unused filesystems](#disable-unused-filesystems)
  - [Enable audit logging](#enable-audit-logging)
  - [Firewall baseline](#firewall-baseline)
  - [Install wazuh-agent](#install-wazuh-agent)

## 1. Create boot disk

    sudo dd if=archlinux.iso of=/dev/sdX bs=4M status=progress oflag=sync

## 2. WIFI Setup

    iwctl
    device list
    station wlan0 scan
    station wlan0 get-networks
    station wlan0 connect WIFI_NAME
    exit

## 3. Prepare disks

    lsblk
    fdisk /dev/nvme0n1

## 3.1 Partitioning

EFI: 

    n
    1
    +512M
    t
    1

ROOT:

    n
    2
    (enter)
    (enter)

SAVE:

    w

## 4. Formating

    mkfs.fat -F32 /dev/nvme0n1p1

## 4.1 Create LUKS encrypted container

    cryptsetup luksFormat /dev/nvme0n1p2
    cryptsetup open /dev/nvme0n1p2 cryptroot
    mkfs.ext4 /dev/mapper/cryptroot

## 5. Mounting

    mount /dev/mapper/cryptroot /mnt
    mkdir /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot

## 6. Install minimal base system

Update package database:

    pacman -Sy

Install:

    pacstrap /mnt base linux linux-firmware nano vim sudo networkmanager git --disable-download-timeout

## 7. Generate fstab

    genfstab -U /mnt >> /mnt/etc/fstab
    cat /mnt/etc/fstab

## 8. Enter new system

    arch-chroot /mnt

## 9. Set timezone

    ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
    hwclock --systohc

## 10. Configure locale

    nano /etc/locale.gen

Uncomment:

    en_US.UTF-8 UTF-8

Generate locale:

    locale-gen

Create:

    nano /etc/locale.conf
    LANG=en_US.UTF-8

## 11. Set hostname

    nano /etc/hostname

Set new hostname:

    archlinux

Open:

    nano /etc/hosts

Add:

    127.0.0.1 localhost
    ::1       localhost
    127.0.1.1 archlinux.localdomain archlinux

## 12. Add local user

    useradd -m -G wheel -s /bin/bash new-user
    passwd new-user

## 13. Enable sudo

    EDITOR=nano visudo

Uncomment:

    %wheel ALL=(ALL:ALL) ALL

## 14. Install bootloader
### 14.1 Configure mkinitcpio for LUKS

    nano /etc/mkinitcpio.conf

Add modules (amd):

    MODULES=(usbhid hid hid_generic atkbd i8042 xhci_hcd xhci_pci ehci_hcd uhci_hcd)

Add modules (nvidia):

    MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)

Add hooks:

    HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)


### 14.2 UUID of encrypted partition
Get uuid of partition:

    blkid /dev/nvme0n1p2

Example output:

    /dev/nvme0n1p2: UUID="aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" TYPE="crypto_LUKS"

### 14.3 Installation
For Nvidia GPU install following packages:

    sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings linux-headers

Systemd-boot:

    bootctl install

    nano /boot/loader/loader.conf

Add:

    default arch.conf
    timeout 3
    editor no

    nano /boot/loader/entries/arch.conf

Add:
    title   Arch Linux
    linux   /vmlinuz-linux
    initrd  /amd-ucode.img
    initrd  /initramfs-linux.img
    options rd.luks.name=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee=cryptroot root=/dev/mapper/cryptroot rw

Additional for nvidia:

    nvidia_drm.modeset=1

Install for AMD CPU:

    pacman -S amd-ucode

### 14.4 Add keyboard

    nano /etc/vconsole.conf
    KEYMAP=de
    FONT=lat9w-16

Install:

    mkinitcpio -P

### Reboot

    sudo reboot
    

# Hardening setup

## Install required security packages

    sudo pacman -S \
    audit \
    fail2ban \
    openssh \
    iptables \
    sudo \
    cronie

Enabled services:

    sudo systemctl enable auditd
    sudo systemctl enable cronie
    sudo systemctl enable sshd

## Kernel hardening

    sudo nano /etc/sysctl.d/99-hardening.conf

Add:

    # IP Spoofing protection
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.default.rp_filter = 1

    # Disable ICMP redirects
    net.ipv4.conf.all.accept_redirects = 0
    net.ipv4.conf.default.accept_redirects = 0
    net.ipv6.conf.all.accept_redirects = 0

    # Disable source routing
    net.ipv4.conf.all.accept_source_route = 0
    net.ipv4.conf.default.accept_source_route = 0

    # Log suspicious packets
    net.ipv4.conf.all.log_martians = 1

    # Protect kernel pointers
    kernel.kptr_restrict = 2

    # Disable SysRq
    kernel.sysrq = 0

    # Restrict dmesg
    kernel.dmesg_restrict = 1

    # Disable unprivileged BPF
    kernel.unprivileged_bpf_disabled = 1

    # Harden BPF JIT
    net.core.bpf_jit_harden = 2

    # FIFO protections
    fs.protected_fifos = 2
    fs.protected_regular = 2

Apply:

    sudo sysctl --system

## Secure SSH

    sudo nano /etc/ssh/sshd_config

Settings:

    PermitRootLogin no
    PasswordAuthentication no
    MaxAuthTries 3
    X11Forwarding no
    ClientAliveInterval 300
    ClientAliveCountMax 2

Apply:

    sudo systemctl restart sshd

## Password policy

Install PAM quality module:

    sudo pacman -S libpwquality

Edit:

    sudo nano /etc/security/pwquality.conf

Policy:

    minlen = 12
    dcredit = -1
    ucredit = -1
    lcredit = -1
    ocredit = -1

## Account lockout - brute force protection

    sudo nano /etc/pam.d/system-auth

Add:

    auth required pam_faillock.so preauth silent audit deny=5 unlock_time=900
    auth [success=1 default=bad] pam_unix.so
    auth [default=die] pam_faillock.so authfail audit deny=5 unlock_time=900
    account required pam_faillock.so

## File permissions

    sudo chmod 600 /etc/shadow
    sudo chmod 644 /etc/passwd
    sudo chmod 600 /etc/gshadow

## Disable unused filesystems

    sudo nano /etc/modprobe.d/disable-filesystems.conf

Add:

    install cramfs /bin/true
    install freevxfs /bin/true
    install jffs2 /bin/true
    install hfs /bin/true
    install hfsplus /bin/true
    install squashfs /bin/true
    install udf /bin/true

## Enable audit logging

    sudo nano /etc/audit/rules.d/hardening.rules

Rules:

    -w /etc/passwd -p wa -k identity
    -w /etc/shadow -p wa -k identity
    -w /etc/group -p wa -k identity
    -w /etc/sudoers -p wa -k sudoers

    -w /var/log/ -p wa -k log_monitor

Apply:

    sudo systemctl restart auditd

## Firewall baseline

    sudo pacman -S ufw
    sudo systemctl enable ufw
    sudo systemctl start ufw

Basic:

    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh

## Install wazuh-agent

    curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg

Add Repo:

    sudo nano /etc/pacman.conf
    [wazuh]
    Server = https://packages.wazuh.com/4.x/arch/

    sudo pacman -S wazuh-agent


