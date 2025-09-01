# ZFS Storage Pools for Slicer

ZFS is an advanced filesystem that was originally developed at [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems). The modern equivalent for Linux was ported as the [OpenZFS project](https://openzfs.org/wiki/Main_Page) and has a license that is incompatible with the Linux Kernel, for that reason, it is kept out of tree, and must be installed separately.

Whilst ZFS can run on a loopback device, this is not recommended and may be unstable. Instead, dedicated either a partition or a drive to running ZFS.

## Use ZFS for VM storage

You'll need to follow the installation instructions below, then you can use the `zfs` type of storage for your VMs.

Take the [walkthrough](/getting-started/walkthrough) example, and change the `storage` type from `image` to `zfs`:

```diff
config:
  host_groups:
  - name: vm
-   storage: image
-   storage_size: 25G
+   storage: zfs
```

Bear in mind that with this initial version of ZFS storage, the VM size will have to match whatever is configured for the zvol snapshotter.

So if the default is not large enough for your needs, delete the snapshot, then adjust the configuration and restart the snapshotter.

## Install packages for ZFS

```bash
sudo apt update
sudo apt install -y zfsutils-linux
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

```
version="0.2.1"
arch="amd64"

wget https://github.com/welteki/zvol-snapshotter/releases/download/v${version}/zvol-snapshotter-${version}-linux-${arch}.tar.gz
sudo tar -C /usr/local/bin \
  -xvf zvol-snapshotter-${version}-linux-${arch}.tar.gz containerd-zvol-grpc

tar -xvf zvol-snapshotter-${version}-linux-${arch}.tar.gz
sudo cp containerd-zvol-grpc /usr/local/bin/
```

Edit `/etc/containerd/config.toml`, add:

```ini
[proxy_plugins]
  [proxy_plugins.zvol]
    type = "snapshot"
    address = "/run/containerd-zvol-grpc/containerd-zvol-grpc.sock"
```

Create a config file for the snapshotter, specifying the size of any VM drive that will be created:

```bash
cat > config.toml <<EOF
root_path="/var/lib/containerd-zvol-grpc"
dataset="slicer/snapshots"
volume_size="30G"
fs_type="ext4"
EOF

sudo mkdir -p /etc/containerd-zvol-grpc
sudo cp config.toml /etc/containerd-zvol-grpc/
```

Create a unit file and install the service:

```bash
cat > ./containerd-zvol-grpc.service <<EOF
[Unit]
Description=zvol snapshotter
After=network.target
Before=containerd.service

[Service]
Type=simple
Environment=HOME=/root
ExecStart=/usr/local/bin/containerd-zvol-grpc --log-level=info --config=/etc/containerd-zvol-grpc/config.toml
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target

EOF

sudo cp ./containerd-zvol-grpc.service /etc/systemd/system/
sudo systemctl enable containerd-zvol-grpc
sudo systemctl start containerd-zvol-grpc
```

Check it worked OK:

```bash
sudo journalctl -u containerd-zvol-grpc -f
```

Reload containerd:

```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd
```

Now, test the Zvol snapshotter:

```bash
(
sudo -E ctr images pull --snapshotter zvol docker.io/library/hello-world:latest
sudo -E ctr run --snapshotter zvol docker.io/library/hello-world:latest test
)
```

## Adjust the base snapshot size

Replace 30G with the desired value i.e. `40G`, then run the following:

```bash
cat > config.toml <<EOF
root_path="/var/lib/containerd-zvol-grpc"
dataset="your-zpool/snapshots"
volume_size="30G"
fs_type="ext4"
EOF

sudo mkdir -p /etc/containerd-zvol-grpc
sudo cp config.toml /etc/containerd-zvol-grpc/
```

Finally reload and restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd-zvol-grpc
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

