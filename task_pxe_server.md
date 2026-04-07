# Task - PXE Server
This task is to showcase the skills to setup a PXE network where devices can boot installation system from network with ks configuration. Structure of the task is as follows

- PXE Boot will have two entires
  - Oracle Linux 8
  - Red Hat 9
- Entries will have ks file for automated installation
  - Each entry will have 3 ks file. Minimal, server, gui to be added to entry manually
- The operating system files will be soft linked from the original source

## 1. Setup DNS and TFTP
We will use dnsmasq since it has both dns server and and tftp server. After installing dnsmasq, we will configure it to serve as the primary dhcp on interface `ens224`

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

## 2. Setup ISO and httpd
Now we setup ISO files and the httpd for the isos to be served to the pxe bootloader