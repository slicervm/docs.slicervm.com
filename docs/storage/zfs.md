# ZFS Storage Pools for Slicer

ZFS is an advanced filesystem that was originally developed at [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems). The modern equivalent for Linux was ported as the [OpenZFS project](https://openzfs.org/wiki/Main_Page) and has a license that is incompatible with the Linux Kernel, for that reason, it is kept out of tree, and must be installed separately.

Whilst ZFS can run on a loopback device, this is not recommended and may be unstable. Instead, dedicated either a partition or a drive to running ZFS.

## Use ZFS for VM storage

Before you can start using `zfs` storage for your VMs, you'll have to run through the installation steps below.

Let's customise the [walkthrough](/getting-started/walkthrough) example for ZFS.

1) Change the `storage` type from `image` to `zfs`:

  ```diff
  config:
    host_groups:
    - name: vm
  -   storage: image
  +   storage: zfs
  -   storage_size: 20G
  ```

  See the note below on `storage_size`.

2) Customise the `storage_size`

  The `storage_size` field is optional for ZFS.

  If not specified, the default size of the base snapshot will be used. A custom size can be given, so long as it is equal to or larger than the base snapshot size.
  
  ```diff
  config:
    host_groups:
    - name: vm
      storage: zfs
  +   storage_size: 40G
  ```

If the base snapshot size is large for any existing VMs, then you can find its lease and remove it before having it re-created with the new settings for the vzol-snapshotter.

```bash
$ sudo ctr -n slicer leases ls
ID            CREATED AT           LABELS 
slicer/k3s-1  2025-09-04T13:53:07Z -      
```

Then, find the lease ID for the VM in question, and delete it. The lease ID is the hostname of the VM i.e.`k3s-1`.

```bash
sudo ctr -n slicer leases rm slicer/k3s-1
```

To delete all leases:

```bash
sudo ctr -n slicer leases ls -q | xargs -n1 sudo ctr -n slicer leases rm
```

## Install packages for ZFS

```bash
sudo apt update -qqy && \
  sudo apt install -qqy --no-install-recommends \
  zfsutils-linux
```

The two commands you are likely to need are:

* `zpool` - create and explore pools to be used by ZFS
* `zfs` - manage ZFS filesystems and snapshots

## Create a pool for Slicer

Let's say that you have a local NVMe i.e. `/dev/nvme1n1`.

Within that device, you may have `/dev/nvme1n1p1` for your operating system, and some free space at `/dev/nvme1n1p2`.

Use the following command to enroll that free space into ZFS:

```bash
sudo zpool create slicer /dev/nvme1n1p2
```

ZFS has many different options such as checksuming, compression, deduplication, encryption, and the ability to run in different RAID-like modes across multiple drives.

We recommend that you do the simplest thing first, to get it working before tinkering further.

## Create a filesystem for Slicer

```bash
sudo zfs create slicer/snapshots
```

## Install the zvol-snapshotter for containerd

The containerd project already has a ZFS snapshotter, however it is unsuitable for use for VMs, therefore we needed to implement our own snapshotter which can present ZFS volumes as block devices to a microVM.

The zvol-snapshotter can be installed using [arkade](https://github.com/alexellis/arkade):

```bash
arkade system install zvol-snapshotter \
  --dataset slicer/snapshots
```

You can specify the size of any VM drive that will be created by the snapshotter using the `--size` flage, e.g. `--size=40G`. The default size is `20G`.

Configure containerd to enable Zvol snapshotter, edit `/etc/containerd/config.toml` and add:

```ini
[proxy_plugins]
  [proxy_plugins.zvol]
    type = "snapshot"
    address = "/run/containerd-zvol-grpc/containerd-zvol-grpc.sock"
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

Check if the snapshotter is running OK:

```bash
sudo journalctl -u zvol-snapshotter -f
```

Now, test the Zvol snapshotter:

```bash
(
sudo -E ctr images pull --snapshotter zvol docker.io/library/hello-world:latest
sudo -E ctr run --snapshotter zvol docker.io/library/hello-world:latest test
)
```

## Adjust the base snapshot size

The base snapshot size can be configured when installing or updating the snapshotter with `arkade system install` by including the `--size` flag.

You can also manually edit the snapshotter configuration file.

Edit `/etc/containerd-zvol-grpc/config.toml` and replace the volume size with the desired value, e.g `40G`:

```diff
root_path="/var/lib/containerd-zvol-grpc"
dataset="your-zpool/snapshots"
-volume_size="30G"
+volume_size="40G"
fs_type="ext4"
```

Finally reload and restart the service:

```bash
sudo systemctl restart zvol-snapshotter
```

## Footnote on ZFS on a loopback file

It is absolutely not recommended to use a loopback file for ZFS for Slicer.

```bash
$ fallocate -l 250G zfs.img
$ sudo zpool create slicer $HOME/zfs.img

zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
slicer   248G   105K   248G        -         -     0%     0%  1.00x    ONLINE  -

$ zfs list
NAME     USED  AVAIL     REFER  MOUNTPOINT
slicer   105K   240G       24K  /slicer
```
