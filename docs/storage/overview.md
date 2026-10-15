# Storage for Slicer

In the [walkthrough](/getting-started/walkthrough), we show how to use disk images for storage.

Disk images were not the first choice for Slicer, initially, only snapshotting filesystems were supported. With a snapshot, the initial filesystem is "unpacked" once - taking anywhere between 30 and 90 seconds, then can be cloned instantly.

* Disk Images - a file per VM, copied from a base disk that Slicer prepares once per image
* Snapshotting / Copy On Write (CoW) - dynamic filesystems that only store changes and allow for instant cloning

## Disk images

Disk images are similar to loopback filesystems which you may have created in the past via `fallocate` and `mkfs.ext4`.

The first VM launched from an image builds a base disk, which is kept in `.slicer/image-base` in the directory you started Slicer from. Every VM after that gets its own copy of the base disk. `slicer image wipe` removes the base disks along with the images.

Pros:

* Convenient, with a custom disk size
* Easy to manage and migrate between systems
* No extra disk, partition, or setup

Cons:

* On ext4, each VM's disk is a full copy of the base disk. This adds a few hundred milliseconds to each launch, and more when many VMs launch at once
* No deduplication on ext4, as is possible with snapshots/Copy On Write (CoW) systems

### Use XFS for faster launches

If the directory you run Slicer from is on XFS, each copy is a reflink: the new disk shares its blocks with the base disk until the VM writes to them. Copying takes a few milliseconds whatever the size of the disk, and VMs only use extra space for what they change.

On a test host, with the default image, moving from ext4 to XFS took each VM's disk preparation from about 430ms to under 15ms, and the time from launch to ready from about 870ms to about 490ms. Ten VMs launched at once were all ready within about 0.7s.

Reflinks need XFS to have been created with them enabled, which is the default for `mkfs.xfs` on current distributions. Check with the mount point of the filesystem you run Slicer from:

```bash
xfs_info /mnt/slicer | grep -o "reflink=[01]"
```

`reflink=1` means copies will be instant.

If you don't have a spare disk or partition, an XFS filesystem in a file works too:

```bash
sudo truncate -s 100G /var/lib/slicer-xfs.img
sudo mkfs.xfs /var/lib/slicer-xfs.img
sudo mkdir -p /mnt/slicer
sudo mount -o loop /var/lib/slicer-xfs.img /mnt/slicer
```

Then run Slicer from a directory under `/mnt/slicer`. Add the mount to `/etc/fstab` if you want it to survive a reboot.

### Resizing disk images

If you need more storage space in a VM using image storage mode, resize the disk after shutting down the VM:

1. **Shutdown the VM**:
   ```bash
   slicer vm shutdown vm-hostname
   ```

   Then close Slicer with Control + C, so you can boot the VM again on next start up.

2. **Check the filesystem** on the disk image:
   ```bash
   sudo e2fsck -fy ./vm-hostname.img
   ```

3. **Resize the filesystem** to a given size i.e. 30G:
   ```bash
   sudo resize2fs ./vm-hostname.img 30G
   ```

4. **Start the VM again**:
   ```bash
   slicer up ./config.yaml
   ```

**Note**: The VM must be using image storage mode (set with `storage: image` in your config). This assumes you've already increased the physical disk image size beforehand.

**Warning**: Always run `e2fsck` before resizing and ensure the VM is completely shut down to avoid data corruption.

## ZFS

[ZFS](https://en.wikipedia.org/wiki/ZFS) is an advanced filesystem that supports CoW snapshots.

Pros:

* Built-in support for snapshots and deduplication
* Instant clone of base snapshot - ideal for launching many VMs
* Easier to troubleshoot and understand than devmapper

Cons:

* More complex to set up and manage than disk images
* Requires an additional disk or partition
* Additional setup required
* No custom sizing - must match the base snapshot

[Setup ZFS storage for Slicer](/storage/zfs)


## Devmapper

[Devmapper](https://en.wikipedia.org/wiki/Device_mapper) is available for Slicer, but not set up by default and requires additional setup.

Generally, we'd recommend using ZFS instead of devmapper unless you have a specific need for it.

Pros:

* Reasonably well known from the Docker space
* Instant clone of base snapshot

Cons:

* Requires an additional disk or partition
* No custom size for VMs - the size of any VM must match the base snapshot
* Difficult to debug and troubleshoot - it's easier to recreate the whole storage pool

[Setup devmapper storage for Slicer](/storage/devmapper)
