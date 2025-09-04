# Devmapper storage for Slicer

[Devmapper](https://en.wikipedia.org/wiki/Device_mapper) is one of the options for storage with Slicer.

We generally recommend using disk images for long-running VMs, or ZFS for launching many short-lived VMs.

That said, you can install Devmapper as an alternative, with a backing drive set up for it, just like we do for ZFS.

## Installation

The setup for Devmapper is different to ZFS, you will need to run the installation script again, but this time, with additional arguments.

First, install a drive, or make a partition available for Devmapper to use.

Run `lsblk` to identify your drives.

```bash
$ lsblk
NAME                             MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme1n1                          259:0    0 931.5G  0 disk 
├─nvme1n1p1                      259:1    0     1G  0 part /boot/efi
├─nvme1n1p2                      259:2    0     2G  0 part /boot
└─nvme1n1p3                      259:3    0 928.5G  0 part 
  └─ubuntu--vg-ubuntu--lv        253:0    0   500G  0 lvm  /
nvme0n1                          259:4    0   1.8T  0 disk 
```

In this instance, you can see that my 2TB NVMe SSD is called nvme0n1 and is currently not allocated.

Head over to the [installation page](/getting-started/install) and run the installation script again, this time include the `VM_DEV` environment variable.

The default size for any unpacked VM is `30GB`, so if you want to alter that size do it now.

```bash
(
cd agent
BASE_SIZE=35GB VM_DEV=/dev/nvme0n1 sudo -E ./install.sh
)
```

Be very careful that you specify the correct drive or partition. This operation cannot be reserved, and will destroy any existing contents.

## Configure Slicer to use Devmapper

```diff
config:
  host_groups:
  - name: vm
-   storage: image
-   storage_size: 25G
+   storage: devmapper
```

The `storage_size` field cannot be specified for Devmapper. So whatever size was setup for snapshots during the installation will be the size used for all VMs.

