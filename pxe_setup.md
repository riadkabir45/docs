# PXE Setup

PXE boot is a to boot operating system to a device without dependency of a storage device. Its a great way to install operating systems in multiple PC from a central data server quickly.

The components of a PXE setup is as follows,

- A `dhcp` server that listens shares to network devices about PXE information

- A `tftp` server to host boot-loader

- A basic file server to host the operating system content publicly

- An ISO or operating system to host on the network

  > **TFTP**: Trivial FIle Transfer Protocol that serves files without any security measurement. Essential for basic boot-loader like PXE to grab files

## 1. Component Installation and Basic Structure

### 1.1 Packages

The basic packages we need are `dhcp`, `tftp`, `syslinux` and `apache2`. Since `dnsmasq` is easier to configure and includes `tftp` built-in, we will use that.

### 1.2 Directory Structure

We mainly need to directory. A sub-folder in `/var/www/html` to host the complete operating system or ISO. And directory containing the basic boot-loader contents to be hosted on `tftp`

```bash
mkdir -p /var/lib/tftpboot
mkdir -p /var/www/html/tftpboot # Our chosen subfolder
```

This part of the the process is messy. As Fedora has everything together in one place while Ubuntu has them spread across directory and package. So lets just learn the basic file needed and locate them distribution wise. They need to be placed in `tftp` content folder.

| Files                   | Purpose                       |
| ----------------------- | ----------------------------- |
| pxelinux.0/syslinux.efi | Entry point basic boot loader |
| ldlinux.c32             | Boot loader required library  |
| menu.c32                | Text menu module              |
| libutil.c32             | Menu function library         |

Now we copy operating system content to `/var/www/html/tftpboot`. For ISO, we just loop mount it.

```bash
mount -o loop PATH_TO_ISO /var/www/html/tftpboot
```

## 2. Configuration

Lets also add the ISO to the **File** `fstab`

```bash
PATH_TO_ISO    /var/www/html/tftpboot    iso9660    loop,ro    0 0
```

Now we configure the `dnsmasq` to listen for any `dhcp` request and send them information about PXE requests.

```ini
# Interface your server uses (e.g., enp0s3)
interface=INTERFACE
bind-interfaces

# PROXY DHCP: Tells clients "I'm a PXE server, but get your IP from the router"
dhcp-range=NETWORK_ADDRESS,proxy

# PXE Logic
dhcp-boot=BOOT_ENTRY_FILE_NAME
enable-tftp
tftp-root=/var/lib/tftpboot
```

> If we were to use our system as the primary `dhcp`, we would change `dhcp-range` to `NETWORK_START_RANGE, END_RANGE, SUBNET, LEASE_TIME`

Now we create a boot menu to be shown in the net-boot which is the **File:**`/var/lib/tftpboot/pxelinux.cfg/default`

```bash
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
MENU TITLE PXE Boot Menu

LABEL BASE_LABEL
    MENU LABEL LINUX_LABEL
    KERNEL vmlinuz
    APPEND initrd=initrd.img inst.repo=http://SERVER_ADDRESS/tftpboot
```

> Note that we might need additional kernel parameter in order to run the system from a network source. Like Oracle Linux needs `devfs=nomount`

Now we start the services `apache2` and `dnsmasq`. Any devices connected to the server via the interface should be able to boot from PXE.

## 3. KS configuration

In order to automate installation, Kick-start configuration is necessary. This will fill out all the options you would need to submit during graphical installation and it supports post install configuration. Already generated KS file can be found in any anaconda used red hat based Linux distribution at `/root/anaconda-ks.cfg`. This file is to be passed as kernel parameter as `inst.ks=URL_PATH_TO_KS`

A sample configuration of KS **File:**

```bash
# Oracle Linux 8 Universal Deployment Kickstart

# Use text mode for a faster, more reliable PXE install
text
# Automatically reboot after the installation is finished
reboot
# Agree to the EULA without manual intervention
eula --agreed

# --- Installation Source ---
# Replace 192.168.2.1 with your Ubuntu Laptop's actual Ethernet IP
url --url="http://192.168.2.1/tftpboot"
repo --name="AppStream" --baseurl="http://192.168.2.1/tftpboot/AppStream"

# --- System Language and Keyboard ---
keyboard --vckeymap=us --xlayouts='us'
lang en_US.UTF-8
timezone Asia/Dhaka --utc

# --- Networking ---
# 'device=link' finds the first plugged-in Ethernet cable automatically
network --bootproto=dhcp --device=link --activate --onboot=on --hostname=setuphost

# --- User Configuration ---
# Use 'openssl -6 PASSWORD' to generate password hash
rootpw --lock
user --groups=wheel --name=USER --password=HASHED_PASSWORD --iscrypted --gecos="USER"

# --- Partitioning (Universal Fix) ---
# 'zerombr' and 'clearpart --all' ensures it wipes whatever disk is found (SATA, NVMe, or USB)
zerombr
clearpart --all --initlabel
autopart

# --- Packages ---
# Using the standard Server environment
%packages
@^server-product-environment
kexec-tools
# Add any extra packages you want installed on all 10 PCs here:
vim
wget
curl
%end

# --- Add-ons ---
%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

# --- Post-Installation Script ---
%post
# You can add commands here to run after the OS is installed
echo "Deployment Successful" > /etc/motd
%end
```

> **Note:** Different base distribution have different method of automation language and passing method. Ubuntu uses Subiquity for example. So individual distribution needs to be researched.
