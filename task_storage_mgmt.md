# Task - Storage Management

This is the the document of completing the project for showcasing basic storage knowledge.

## 1. Disk Identification

The used vm instance has following block tree

```bash
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                      11:0    1  9.5G  0 rom  
nvme0n1                 259:0    0   10G  0 disk 
├─nvme0n1p1             259:1    0   94M  0 part 
└─nvme0n1p2             259:2    0   48M  0 part 
nvme0n2                 259:3    0   20G  0 disk 
├─nvme0n2p1             259:6    0    1M  0 part 
├─nvme0n2p2             259:7    0    1G  0 part /boot
└─nvme0n2p3             259:8    0   19G  0 part 
  ├─ol_tabiulhasan-root 252:0    0   17G  0 lvm  /
  └─ol_tabiulhasan-swap 252:1    0    2G  0 lvm  [SWAP]
nvme0n4                 259:4    0   10G  0 disk 
nvme0n3                 259:5    0   10G  0 disk 
```

The disk which is used by OS is nvme0n2 was named nvme0n1 before reboot. The following disk nvme0n1, nvme0n4, nvme0n3 are later added to show case storage management.

For the following task, nvme0n1 will be reused and wiped clean. Since name has been noticed to change for some reason during reboot, UUID will be used instead.

## 2. Standard Partition Configuration

In this task, we will format the whole disk with XFS, mount it and make the mount persistent.

Lets create recreate storage structure

```bash
# Recreate the disk partition table with gpt
parted /dev/nvme0n1 mktable gpt

# Get disk stats
parted /dev/nvme0n1 print

# Create a parition named ProjectData to take full space
parted /dev/nvme0n1 mkpart 'ProjectData' 2048s 10.7G

# Format the disk to xfs file system
mkfs.xfs /dev/nvme0n1p1

# Create the mount point
mkdir /projectdata

# Mount the xfs
mount /dev/nvme0n1p1 /projectdata

# Grab UUID of the parittion
blkid | greo n1p1
```

Current state of `/etc/fstab`

```bash
UUID=d325996e-c1a9-4452-9379-ddc0bd762994 /                       xfs     defaults        0 0
UUID=cca2859c-af8d-484b-8296-d2ba53eb0bd3 /boot                   xfs     defaults        0 0
UUID=6001f0f4-d1cb-475c-a556-7f61f5efbda0 none                    swap    defaults        0 0
```

State after change

```bash
UUID=d325996e-c1a9-4452-9379-ddc0bd762994 /                       xfs     defaults        0 0
UUID=cca2859c-af8d-484b-8296-d2ba53eb0bd3 /boot                   xfs     defaults        0 0
UUID=6001f0f4-d1cb-475c-a556-7f61f5efbda0 none                    swap    defaults        0 0
UUID="4c7d1fe7-fa8b-4cb0-adb3-ac2a99b6ff24" /projectdata            xfs     defaults        0 0
```

## 3. LVM setup

The device names again has changed after reboot. Target device for this task is `nvme0n4` which is now named `nvme0n3`

```bash
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                      11:0    1  9.5G  0 rom  
nvme0n1                 259:0    0   10G  0 disk 
└─nvme0n1p1             259:1    0   10G  0 part /projectdata
nvme0n2                 259:2    0   10G  0 disk 
└─nvme0n2p1             259:6    0   10G  0 part 
  ├─vg_training-lv_data 252:2    0    6G  0 lvm  /data
  └─vg_training-lv_logs 252:3    0    4G  0 lvm  /logs
nvme0n4                 259:3    0   20G  0 disk 
├─nvme0n4p1             259:5    0    1M  0 part 
├─nvme0n4p2             259:7    0    1G  0 part /boot
└─nvme0n4p3             259:8    0   19G  0 part 
  ├─ol_tabiulhasan-root 252:0    0   17G  0 lvm  /
  └─ol_tabiulhasan-swap 252:1    0    2G  0 lvm  [SWAP]
nvme0n3                 259:4    0   10G  0 disk 
```

After creating partition and preparing it for lvm

```bash
vgcreate vg_training /dev/nvme0n3p1
```

Then create the logical volumes from the group

```bash
lvcreate -L 6G vg_training -n lv_data
lvcreate -l 100%FREE vg_training -n lv_logs
```

Format the new devices

```bash
mkfs.xfs /dev/mapper/vg_training-lv_data
mkfs.xfs /dev/mapper/vg_training-lv_logs
```

Create and mount on the directories

```bash
mkdir /data /logs
mount /dev/mapper/vg_training-lv_data /data
mount /dev/mapper/vg_training-lv_data /logs
```

Running `vgs` yields the following free space result

```bash
  VG             #PV #LV #SN Attr   VSize   VFree
  ol_tabiulhasan   1   2   0 wz--n- <19.00g    0 
  vg_training      1   2   0 wz--n- <10.00g    0 
```

An extra command is ran to make blkid cache reload

```bash
blkic -c /dev/null
```

For persistent mount

```bash
UUID=d325996e-c1a9-4452-9379-ddc0bd762994 /                       xfs     defaults        0 0
UUID=cca2859c-af8d-484b-8296-d2ba53eb0bd3 /boot                   xfs     defaults        0 0
UUID=6001f0f4-d1cb-475c-a556-7f61f5efbda0 none                    swap    defaults        0 0
UUID=4c7d1fe7-fa8b-4cb0-adb3-ac2a99b6ff24 /projectdata            xfs     defaults        0 0
UUID="a7fcbfaf-ff6e-4760-8f5f-c70c86e4161d" /data                  xfs      defaults          0 0
UUID="f3d257cd-5e53-4a20-8811-8078e7e1833d" /logs                  xfs      defaults          0 0
```

## 4. Logical Volume Extension

First prep another partition for LVM

```bash
pvcreate /dev/nvme0n3p1
```

Use the partition to extend the volume group

```bash
vgextend vg_training /dev/nvme0n3p1
```

Now extend the logs volume by 6G

```bash
lvextend -L +5G /dev/mapper/vg_training-lv_data
```

Now make /data take up the new space

```bash
xfs_growfs /data
```

For verification we use `df -h /data`

```bash
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/vg_training-lv_data   11G  248M   11G   3% /data
```

## 5. Insufficient Space

Trying to add 6GB to `lv_data` we get

```bash
lvextend -L +6G /dev/mapper/vg_training-lv_logs
#-> Insufficient free space: 1536 extents needed, but only 1279 available
```

So lets add our new disk `nvme0n5` to the group and extend with lvextend

```bash
pvcreate /dev/nvme0n5p1
vgextend vg_traininig /dev/nvme0n5p1
lvextend -L +6G /dev/mapper/vg_training-lv_logs
sudo xfs_growfs /logs/

df -h /logs/
#-> Filesystem                       Size  Used Avail Use% Mounted on
#-> /dev/mapper/vg_training-lv_logs   10G  228M  9.8G   3% /logs
```

## Create Logical Volume Using Remaining Space

To create a volume with remaining space we use

```bash
lvcreate -l 100%free vg_training -n lv_backup
```

For the rest

```bash
mkfs.xfs /dev/mapper/vg_training-lv_backup 
```

Updated fstab

```bash
UUID=d325996e-c1a9-4452-9379-ddc0bd762994 /                       xfs     defaults        0 0
UUID=cca2859c-af8d-484b-8296-d2ba53eb0bd3 /boot                   xfs     defaults        0 0
UUID=6001f0f4-d1cb-475c-a556-7f61f5efbda0 none                    swap    defaults        0 0
UUID=4c7d1fe7-fa8b-4cb0-adb3-ac2a99b6ff24 /projectdata            xfs     defaults        0 0
UUID=a7fcbfaf-ff6e-4760-8f5f-c70c86e4161d /data                      xfs      defaults          0 0
UUID=f3d257cd-5e53-4a20-8811-8078e7e1833d /logs                      xfs      defaults          0 0
UUID="608e69de-fcb4-4122-9857-5cb9d8eb1f8a" /backup               xfs     defaults        0 0
```

## 6. Discard backup volume

First lets unmount the partition

```bash
umount /backup
```

Then remove it from the fstab file

```bash
UUID=d325996e-c1a9-4452-9379-ddc0bd762994 /                       xfs     defaults        0 0
UUID=cca2859c-af8d-484b-8296-d2ba53eb0bd3 /boot                   xfs     defaults        0 0
UUID=6001f0f4-d1cb-475c-a556-7f61f5efbda0 none                    swap    defaults        0 0
UUID=4c7d1fe7-fa8b-4cb0-adb3-ac2a99b6ff24 /projectdata            xfs     defaults        0 0
UUID=a7fcbfaf-ff6e-4760-8f5f-c70c86e4161d /data                     xfs     defaults     0 0
UUID=f3d257cd-5e53-4a20-8811-8078e7e1833d /logs                     xfs     defaults     0 0
#UUID="608e69de-fcb4-4122-9857-5cb9d8eb1f8a" /backup               xfs     defaults        0 0
```

Now safely delete the volume and return the space to group

```bash
lvremove /dev/mapper/vg_training-lv_backup
```

Verification of returned space uing `vgs`

```bash
  VG             #PV #LV #SN Attr   VSize   VFree
  ol_tabiulhasan   1   2   0 wz--n- <19.00g    0 
  vg_training      3   2   0 wz--n- <29.99g 8.99g
```

## 8. NFS configuration

First install and enable the NFS service under package `nfs-utils`

```bash
# Install nfs
dnf install nfs-utils

# Enable nfs
systemctl enable nfs-server --now
```

Add the folder for exports in `/etc/exports`

```bash
# Dir        Allowed IP(Options)
/data        192.168.2.0/24(rw)
```

Tell service to reload exports using

```bash
exportfs -arv
```

Mount it and test from another system

```bash
mount -t nfs 192.168.2.133:/data /mnt/share
```

> Sometimes host can have firewall blocks that deny external access. Make sure to open them with `firewall-cmd --add-service={nfs,rpc-bind,mountd} --permanent`

Create fstab entry for persistent mount

```bash
192.168.2.133:/data    /mnt/share    nfs    defaults    0 0
```
