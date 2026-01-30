# Storage for Slicer

In the [walkthrough](/getting-started/walkthrough), we show how to use disk images for storage.

Disk images were not the first choice for Slicer, initially, only snapshotting filesystems were supported. With a snapshot, the initial filesystem is "unpacked" once - taking anywhere between 30 and 90 seconds, then can be cloned instantly.

* Disk Images - static files allocated on start-up, can take a few seconds to clone the initial disk
* Snapshotting / Copy On Write (CoW) - dynamic filesystems that only store changes and allow for instant cloning

## Disk images

Disk images are similar to loopback filesystems which you may have created in the past via `fallocate` and `mkfs.ext4`.

Pros:

* Convenient, with a custom disk size
* Easy to manage and migrate between systems

Cons:

* No deduplication, as is possible with snapshots/Copy On Write (CoW) systems
* Launch/clone time slower than CoW when launching many VMs

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
