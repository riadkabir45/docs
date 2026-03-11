# Task - RAID System

Create and examine robustness of RAID system by doing the following actions,

- Create 2 disk RAID

- Copy a file into them

- Destroy 1st disk

- Attach 3rd disk

- Destroy 2nd disk

- Generate sha256 checksum of file from 3rd disk

- Verify checksum

## 1. Create two RAID disks

First we need to partition two disks. Ensure the partitions are same size for RAID to work.  For this task we will create each partition with 4916MB size.

> Doing RAID on raw disk is not recommended as even same model disk are not always same in size.

Next we will create the RAID system and add those two disks

```bash
mdadm --create --level=1 --raid-devices=2 /dev/md45 /dev/nvme0n1p1 /dev/nvme0n3p1
```

We can now see our RAID system is active by looking in `/proc/mdstat`

```bash
Personalities : [raid1] 
md45 : active raid1 nvme0n3p1[1] nvme0n1p1[0]
      5028864 blocks super 1.2 [2/2] [UU]
      bitmap: 0/1 pages [0KB], 65536KB chunk
```

Now we can format the RAID disk and mount it which is `dev/md45`

```bash
# Format the disk
mkfs.ext4 /dev/md45
# Mount it
mount /dev/md45 /mnt/RAID
```

Now we copy a large flle. For this experiment we chose mint iso which has checksum in sha256

Name:         `linuxmint-22.3-cinnamon-64bit.iso`

Checksum: `a081ab202cfda17f6924128dbd2de8b63518ac0531bcfe3f1a1b88097c459bd4`

Now we mark a disk as failed and remove it

```bash
# Mark it failed
mdadm /dev/md45 --fail /dev/nvme0n3p1

# Remove the disk
mdadm /dev/md45 --remove /dev/nvme0n3p1

# Both can be done in one line
mdadm /dev/md45 --fail /dev/nvme0n3p1 --remove /dev/nvme0n3p1
```

Now `/proc/mdstat` will show following degraded disk status

```bash
md45 : active raid1 nvme0n3p1[1]
      5028864 blocks super 1.2 [2/1] [_U]
      bitmap: 1/1 pages [4KB], 65536KB chunk
```

Now we add out 3rd disk which also has been partitioned to same size

```bash
mdadm /dev/md45 --add /dev/sda1
```

Now `/proc/mdstat` will show a recovering progress

```bash
Personalities : [raid1] 
md45 : active raid1 sda1[2] nvme0n3p1[1]
      5028864 blocks super 1.2 [2/1] [_U]
      [===>.................]  recovery = 19.9% (1001984/5028864) finish=0.2min speed=250496K/sec
      bitmap: 0/1 pages [0KB], 65536KB chunk
```

After recovery is finished, we also remove the 2nd disk `nvme0n3p1`

After which sha256 checksum is

```bash
a081ab202cfda17f6924128dbd2de8b63518ac0531bcfe3f1a1b88097c459bd4
```

## 2. Check robustness with data corruption

Now we add the disk back, let it recover and apply a disk massive write damage on one disk.

```bash
sudo dd if=/dev/zero of=/dev/nvme0n3p1 bs=1024M count=3 status=progress
```

Now lets tell system to do a recovery

```bash
echo check | sudo tee /sys/block/md45/md/sync_action
```

After recovery, the checksum is

```bash
53520dc09bbfc2f7c7c8f2a550941b0255a12b4f7831929c46847c07cf68aea2
```

So, RAID1 cannot survive recovery from massive disk write corruption.
