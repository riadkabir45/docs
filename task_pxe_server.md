# Task - PXE Server
This task is to showcase the skills to setup a PXE network where devices can boot installation system from network with ks configuration. Structure of the task is as follows

- PXE Boot will have two entires
  - Oracle Linux 8
  - Red Hat 9
- Entries will have ks file for automated installation
  - Each entry will have 3 ks file. Minimal, server, gui to be added to entry manually
- The operating system files will be soft linked from the original source

## 1. Setup DNS and TFTP
We will use dnsmasq since it has both dns server and and tftp server. After installing dnsmasq, we will configure it to serve as the primary dhcp on interface `ens224`. Also make sure that our device IP address is static.

**File:** `/etc/dnsmasq.conf`
```ini
interfaces=ens224
bind-dynamic

dhcp-range=192.168.80.50,192,168.80.150,255.255.255.0,1h

enable-tftp
tftp-root=/opt/tftpboot
dhcp-boot=pxelinux.0
```

> **Note:** Even though standard documentations use `bind-interfaces` instead of `bind-dynamic`.
> The 2nd configuration is better when multiple interfaces cause some to start slow.
> As `bind-interfaces` is very strict about interface being ready from start. 

Now we will setup the tftp folder
```bash
sudo mkdir -p /opt/tftpboot/pxelinux.cfg
sudo cp /usr/share/syslinux/{ldlinux.c32,libutil.c32,menu.c32,pxelinux.0} /opt/tftpboot/

cat << EOF > /opt/tftpboot/pxelinux.cfg/default

DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
MENU TITLE PXE Boot Menu

LABEL BASE_LABEL
    MENU LABEL Oracle 8
    KERNEL boot/ol/vmlinuz
    APPEND initrd=boot/ol/initrd.img devfs=nomount inst.repo=http://192.168.80.2/pxe/ol inst.ks=http://192.168.80.2/pxe/olm.cfg

LABEL BASE_LABEL
    MENU LABEL Rhel 9
    KERNEL boot/rhel/vmlinuz
    APPEND initrd=boot/rhel/initrd.img devfs=nomount inst.repo=http://192.168.80.2/pxe/rhel inst.ks=http://192.168.80.2/pxe/rhelg.cfg
EOF
```

> **Note:** We habe to make sure our ip address is static so the IP addresses in the boot menu and ks file align with out device ip on every boot.

## 2. Setup ISO and httpd
Now we setup ISO files and the httpd for the isos to be served to the pxe bootloader. First we place the iso to a specific location such as `/opt/iso`. And mounth them to `/var/www/html/pxe/ol` `/var/www/html/pxe/rhel` respectively. We can mount them via `fstab`.

**File:** `/etc/fstab`
```bash
/opt/iso/ol8u10_x86_64.iso                /var/www/html/pxe/ol      iso9660 loop,ro         0 0
/opt/iso/rhel-9.7-x86_64-dvd.iso          /var/www/html/pxe/rhel    iso9660 loop,ro         0 0
```

Now we run `mount -a` and if no error is present, they should be mounted. Then we create a symlink to `./isoliux` folder of both iso in `/opt/tftpboot/boot` folder.
```bash
ln -sf /var/www/html/pxe/ol/isolinux/ /opt/tftpboot/boot/ol
ln -sf /var/www/html/pxe/rhel/isolinux/ /opt/tftpboot/boot/rhel
```

## 3. Generate KS File
Now we prep out ks files. We will make 3 ks files for each iso. Those are minimal, server and gui. Red hat provides a gui tool to configure ks files. But for this experiment we will modify an existing ks file from manual installation to make our ks file

**Original KS File:** This is for minimal server installation
```conf
#version=OL8
# Use graphical install
graphical
reboot

repo --name="AppStream" --baseurl=http://192.168.80.10/pxe/ol/AppStream

%packages
@^minimal-environment
kexec-tools

%end

# Keyboard layouts
keyboard --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network --bootproto=dhcp --device=link --activate --onboot=on --hostname=ol8-gui

# Use network installation
url --url="http://192.168.80.10/pxe/ol"

# Run the Setup Agent on first boot
firstboot --enable

zerombr
clearpart --all --initlabel
autopart

# System timezone
timezone Asia/Dhaka --isUtc

# Root password
rootpw --lock
user --name=riad --password=$6$anOMAV5FE00AV.3r$C.B1Q6wOT9BBxszbuhS10zWxuqxKiUejCN51novMIbMr5ZB2xM3gpnkjTjRE.slnMQ8CwlP1s4q8mbZo98m8V0 --iscrypted --gecos="riad"

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end
```

Here we will changet the user config to a generic default password that is not encrypted
```conf
user --name=riad --password=password --iscrypted --gecos="riad"
```

And to primpt user to change password on first login. We will add this to end
```conf
%post
chage -d 0 riad
%end
```

And for server and GUI ks file, we will change package section to this respectively
```conf
%packages
@^server-product-environment
kexec-tools
%end
```

```conf
%packages
@^graphical-server-environment
kexec-tools
%end
```

## 4. Enable All Services
Now we enable dnsmasq and httpd so that our device works as a dns server and serves a os tree.
```bash
sudo systemctl enable dnsmasq.service --now
sudo systemctl enable httpd.service --now
```bash