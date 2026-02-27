# Installation for Slicer for Linux

!!! note "Slicer for Linux and sudo"
    The Slicer for Linux daemon requires root, and can listen on a UNIX socket or a a TCP port. When it listens on a UNIX socket, then typically, you'll want to run `slicer` as a client using `sudo -E`. The `-E` flag preserves the user's environment variables and HOME directory.

Don't wait for the perfect system. Slicer can run practically anywhere.

To activate Slicer, pick the tier that matches your use-case on the [Slicer pricing page](https://slicervm.com/pricing). A free trial is available.

If you need Slicer for Mac instead, use the [Slicer for Mac installation guide](/mac/installation) first.

After the installation, when you run `slicer activate` you'll get an invite link to the Discord server. We highly recommend joining.

## System requirements

Any reasonably modern computer can run Slicer, the requirements are very low - x86_64 or Arm64 (including the Raspberry Pi).

Ideal for labs and the home office:

* Low powered mini PC i.e. [Intel N100](https://blog.alexellis.io/n100-mini-computer/), Beelink,  Minisforum,  Acemagic, etc
* [Adlink Ampere Developer Platform](https://www.adlinktech.com/Products/computer_on_modules/COM-HPC-Server-Carrier-and-Starter-Kit/Ampere_Altra_Developer_Platform?lang=en) / [system76 Thelio Astra](https://system76.com/desktops/thelio-astra-a1.1-n1/configure) / [Raspberry Pi 4 or 5](https://www.raspberrypi.com/products/raspberry-pi-5/) (an NVMe is better than SD card)
* Mac Mini M1 or M2 (with [Asahi Linux](https://asahilinux.org/) installed)
* PC, laptop, or used server from eBay - under your desk or in your basement.

Cloud-based bare-metal:

* [Hetzner bare-metal](https://www.hetzner.com/bare-metal-server) aka "robot" (cheapest, best value)
* [Phoenix NAP](https://phoenixnap.com/bare-metal-cloud/instances)
* [Latitude.sh](https://www.latitude.sh/features)

Additional cloud-based options for KVM are [included on this page on our sister site (Actuated)](https://docs.actuated.com/provision-server/)

Enterprise:

* On-premises datacenter with your own bare-metal servers
* OpenStack / VMware (with nested virtualisation)
* Azure, DigitalOcean, GCP VMs (with nested virtualisation)

A Linux system with KVM is required (bare-metal or nested virtualisation), so if you see `/dev/kvm`, Slicer will work there.

Ubuntu LTS is formally supported, whilst Debian, Fedora, RHEL-like Operating Systems (Rocky, Alma, CentOS), and Arch Linux should work - we won't be able to debug your system.

Ideally, nothing else should be installed on a host that runs Slicer. It should be thought of as a basic appliance - a bare OS, with minimal packages.

## Quick installation

The default slicer installation only enables support for [`image` storage](/storage/overview). Additional storage backends for [zfs](/storage/zfs) or [devmapper](/storage/devmapper) can be enabled using the `--zfs` and `--devmapper` flags. See [Snapshot-based storage](#snapshot-based-storage).

```sh
curl -sLS https://get.slicervm.com | sudo bash
```

The installer sets up [Firecracker](https://firecracker-microvm.github.io), [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor), [containerd](https://containerd.io/) for storage, and a few networking options.

> Feel free to read/verify [the installation script](https://get.slicervm.com) before running it.

Additional storage backends can always be enabled later by running the installer again with the appropriate flags.

Activate Slicer to obtain your license key:

```bash
slicer activate --help

slicer activate
```

For the Individual tier the a license key will be written to `~/.slicer/LICENSE` along with a GitHub token. The key will last for 30 days, but can be refreshed via `slicer activate`. When refreshing, it'll use the stored GitHub token, instead of running the device flow again.

For higher tiers, the license key is an API key delivered to you via email. You should save it to `~/.slicer/LICENSE`.

Next, start your first VM with the [walk through](/getting-started/walkthrough).

## Snapshot-based storage

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

## Updating slicer

To update Slicer, use the `slicer update` command:

```bash
sudo -E slicer update
```

Alternatively, if you're on an earlier version, repeat this command from the installation step:

```bash
sudo -E arkade oci install ghcr.io/openfaasltd/slicer:latest \
  --path /usr/local/bin
```
