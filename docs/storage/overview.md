# Storage for Slicer

In the [walkthrough](/getting-started/walkthrough), we show how to use disk images for storage.

Disk images were not the first choice for Slicer, initially, only snapshotting filesystems were supported. With a snapshot, the initial filesystem is "unpacked" once - taking anywhere between 30 and 90 seconds, then can be cloned instantly.

* Disk Images - static files allocated on start-up, can take a few seconds to clone the initial disk
* Snapshotting / Copy On Write (CoW) - dynamic filesystems that only store changes and allow for instant cloning

The current installation supports Disk Images only, and we will add the instructions for ZFS shortly.

## Disk images

Disk images are similar to loopback filesystems which you may have created in the past via `fallocate` and `mkfs.ext4`.

Pros:

* Convenient, with a custom disk size
* Easy to manage and migrate between systems

Cons:

* No deduplication, as is possible with snapshots/Copy On Write (CoW) systems
* Launch/clone time slower than CoW when launching many VMs

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