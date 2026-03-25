# Installation for Slicer for Linux

Looking for Slicer for Mac? See the [Slicer for Mac installation guide](/mac/installation).

## System requirements

You can run Slicer on a varied range of hardware, from mini PCs, to your Mac, to cloud-based VMs with nested virtualisation support.

* Bare-metal or nested virtualisation
* x86_64 or Arm64 (including Raspberry Pi)
* Ubuntu LTS is the preferred/supported OS (Debian, and RHEL-like should also work)

See the [appendix](#appendix) for suggested local hardware and cloud-based options.

Slicer is designed to run on a basic OS installation, without Docker, Kubernetes, or other virtualisation tools. Slicer installs and manage its own dependencies, such as Firecracker, [QEMU](https://www.qemu.org/), and containerd.

## Quick installation

The default slicer installation only enables support for [`image` storage](/storage/overview). Additional storage backends for [zfs](/storage/zfs) or [devmapper](/storage/devmapper) can be enabled using the `--zfs` and `--devmapper` flags. See [Snapshot-based storage](#snapshot-based-storage).

```sh
curl -sLS https://get.slicervm.com | sudo bash
```

> See also: [installation script](https://github.com/slicervm/slicervm.com/blob/master/get.sh)

The installer sets up [Firecracker](https://firecracker-microvm.github.io), [QEMU](https://www.qemu.org/), [containerd](https://containerd.io/) for storage, and a few networking options.

**Setup the license key**

If you have a subscription for Slicer Individual, Team or Platform, then you'll have received a license key via email. Save it to `~/.slicer/LICENSE`. This license will not expire, so long as your subscription remains active.

If you're paying for Slicer Individual via GitHub Sponsors, then after installation, you should run `slicer activate` to link your GitHub account to your Slicer installation. The keys for sponsors last for 30 days, but can be refreshed using the same command.

Next, start your first VM with the [walk through](/getting-started/walkthrough).

For production, use [snapshot based storage](#snapshot-based-storage) for near-instance VM creation times.

### Updating slicer

To update Slicer, use the `slicer update` command:

```bash
sudo slicer update
```

## Appendix

### Local hardware for Slicer

Ideal for labs/or and the home office:

* Low powered mini PC i.e. [Intel N100](https://blog.alexellis.io/n100-mini-computer/), Beelink,  Minisforum,  Acemagic, etc
* [Adlink Ampere Developer Platform](https://www.adlinktech.com/Products/computer_on_modules/COM-HPC-Server-Carrier-and-Starter-Kit/Ampere_Altra_Developer_Platform?lang=en) / [system76 Thelio Astra](https://system76.com/desktops/thelio-astra-a1.1-n1/configure) / [Raspberry Pi 4 or 5](https://www.raspberrypi.com/products/raspberry-pi-5/) (an NVMe is better than SD card)
* Mac Mini M1 or M2 (with [Asahi Linux](https://asahilinux.org/) installed)
* PC, laptop, or used server from eBay - under your desk or in your basement.

### Cloud options for Slicer

Cloud-based bare-metal:

* [Hetzner bare-metal](https://www.hetzner.com/bare-metal-server) aka "robot" (cheapest, best value)
* [Phoenix NAP](https://phoenixnap.com/bare-metal-cloud/instances)
* [Latitude.sh](https://www.latitude.sh/features)

Enterprise:

* On-premises datacenter with your own bare-metal servers
* OpenStack / VMware (with nested virtualisation)
* Azure, DigitalOcean, GCP VMs (with nested virtualisation)

Additional cloud-based options for KVM are [included on this page on our sister site (Actuated)](https://docs.actuated.com/provision-server/)

### Snapshot-based storage

Snapshot-based storage is not required for development and testing, instead, it's recommended that most users use the `image` storage approach instead, until they want to trade a little extra setup, for much improved VM disk clone times.

Snapshot-based storage enables much faster VM creation times. ZFS is the recommended option for Slicer, devmapper is also supported. See [storage for slicer](/storage/overview) for more info on the different storage backends.

For best performance using a dedicated drive, volume or partition for the storage backend is recommended. If no disk is provided a loopback device will be created automatically.

!!! note "ZFS on non-Ubuntu distributions"
    Automatic ZFS installation is only supported on Ubuntu. On other distributions, [install ZFS manually](https://openzfs.github.io/openzfs-docs/Getting%20Started/index.html) before running the install script with the `--zfs` flag.

**ZFS (loopback)**

```sh
curl -sLS https://get.slicervm.com | sudo bash -s -- \
  --zfs
```

**ZFS (dedicated drive)**

```sh
curl -sLS https://get.slicervm.com | sudo bash -s -- \
  --zfs /dev/sdb
```

**Devmapper (loopback)**

```sh
curl -sLS https://get.slicervm.com | sudo bash -s -- \
  --devmapper
```

**Devmapper (dedicated drive)**

```sh
curl -sLS https://get.slicervm.com | sudo bash -s -- \
  --devmapper /dev/sdb
```

The `--devmapper` and `--zfs` flags can be used together to enable both storage backends.

