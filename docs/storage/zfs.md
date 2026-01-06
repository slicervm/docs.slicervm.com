# ZFS Storage Pools for Slicer

ZFS is an advanced filesystem that was originally developed at [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems). The modern equivalent for Linux was ported as the [OpenZFS project](https://openzfs.org/wiki/Main_Page) and has a license that is incompatible with the Linux Kernel, for that reason, it is kept out of tree, and must be installed separately.

The containerd project already has a ZFS snapshotter, however it is unsuitable for use for VMs, therefore we implemented our own snapshotter plugin which can present ZFS volumes as block devices to a microVM.

## Installation

To setup ZFS you can run the installation script again, but this time, with additional flags.

Install a drive, or make a partition available for zfs to use. The installation script will automatically creatre a loopback device if no device is provided.

While using a loopback file is supported, it is absolutely not recommended to use a loopback file for ZFS with Slicer.

Run the installation script again and set the `--zfs` flag:

```sh
curl -sLS https://get.slicervm.com | sudo bash -s -- \
  --zfs /dev/nvme0n1 \
  --overwrite # Destroy any existing content on the disk
```

Be very careful that you specify the correct drive or partition. This operation cannot be reversed and will destroy any existing contents.

The default size for any unpacked VM disk is `30GB`. See [adjust the base snapshot size](#adjust-the-base-snapshot-size) to change this.

## Use ZFS for VM storage

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

## Adjust the base snapshot size

The base snapshot size can be changed by updating the snapshooter configuration file and restarting the zvol snapshotter service.

Edit `/etc/containerd-zvol-grpc/config.toml` and replace the volume size with the desired value, e.g `40G`:

```diff
root_path="/var/lib/containerd-zvol-grpc"
dataset="your-zpool/snapshots"
-volume_size="30G"
+volume_size="40G"
fs_type="ext4"
```

Finally restart the zvol-snapshotter service:

```bash
sudo systemctl restart zvol-snapshotter
```
