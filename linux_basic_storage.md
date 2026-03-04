# Linux Basic Storage

Partitions improve performance of system and applications and also helps critical files from regular files. There are two kind of partition system:

- MBR: The old traditional partition scheme supported by legacy BIOS

- GPT: The modern scheme that is used in UEFI firmware systems

## MBR - Master Boot Record

MBR initially supports 4 partitions. Using extended partition and logical partitions, one can stretch the partition number to 15. Other limitations are:

- 32bit partition size

- Up to 2 TB disk support

## GPT -  GUID Partition Table

GPT is the replacement of MBR, extending all of its limitations. It uses GUID to create partition identity. It also brings better recovery support by keeping a primary GPT and backup GPT at begin and end of disk respectively. Features are:

- 128 partitions

- Up to 8 ZB storage

- 64 bit partition addresses

---

## Parted - the partition manager I seem to hate

Parted is a disk manager that both helps you and also not help you. It helps you align ends to specific disk requirement but it does not help you auto align starts and ends to maybe most left appropriate start and most appropriate end.

To use parted

```bash
# Open a disk and provide interface to run subcommands
parted /dev/DEVICE_PATH

# Run a sub command on a disk
parted /dev/DEVICE_PATH SUB_COMMAND
```

Common parted subcommands

```bash
# All these command can be run by just calling subcommand name
# and passing each value seperately

# Create parition table 
mklabel PARTITION_TYPE
# Or with
mktable PARITTION_TYPE

# To create parition
mkpart PART_NAME FS_TYPE START END
# Start and end both supports siffx such as KB, MB, GB, KiB, Mib, GiB

# Print current disk details paritions and their ID
print

# Remove a parition
rm PARTITION_ID
```



After any kind of partition change `udevadm settle` might be needed to run for kernel to detect changes

---

# Managing Swap

Swap are disk space used as backup memory to keep things that are not needed currently or needs to survive power off.



```bash
# Prepare a disk or block for swap
mkswap /dev/BLOCK_PATH

# Eanble block as swap file to be used by the system
swapon /dev/BLOCK_PATH

# Disable a swap
swapoff /dev/BLOCK_PATH
```



> Note: Swap space takes mount parameter of pri which defines which swap will be prioritized for using till filled up. The default value is -2
