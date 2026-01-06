# Installation for Slicer

Don't wait for the perfect system. Slicer can run practically anywhere.

To activate Slicer, you'll need a [Home Edition or Commercial subscription](https://slicervm.com/pricing) - pick according to your needs. Both are available on a monthly basis.

After the installation, when you run `slicer activate` you'll get an invite link to the Discord server. We highly recommend joining.

## System requirements

Any reasonably modern computer can run Slicer, the requirements are very low - x86_64 or Arm64 (including the Raspberry Pi).

Ideal for labs and the home office:

* Low powered mini PC i.e. [Intel N100](https://blog.alexellis.io/n100-mini-computer/), Beelink,  Minisforum,  Acemagic, etc
* [Adlink Ampere Developer Platform](https://www.adlinktech.com/en/products/embedded-computing/adlink-ampere-developer-platform) / [Raspberry Pi 4 or 5](https://www.raspberrypi.com/products/raspberry-pi-5/) (an NVMe is better than SD card)
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

Ubuntu LTS is formally supported, whilst Arch Linux and RHEL-like Operating Systems should work - we won't be able to debug your system.

Ideally, nothing else should be installed on a host that runs Slicer. It should be thought of as a basic appliance - a bare OS, with minimal packages.

## Quick installation

The default slicer installation only enables support for [`image` storage](/storage/overview). Additional storage backends for [zfs](/storage/zfs) or [devmapper](/storage/devmapper) can be enabled using the `--zfs` and `--devmapper` flags. See [Snapshot-based storage](#snapshot-based-storage).

```sh
curl -sLS https://get.slicervm.com | sudo bash
```

The installer sets up [Firecracker](https://firecracker-microvm.github.io), [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor), [containerd](https://containerd.io/) for storage, and a few networking options.

> Feel free to read/verify [the installation script](https://get.slicervm.com) before running it.

Additional storage backends can always be enabled later by running the installer again with the appropriate flags.

For Home Edition/Hobbyist users, activate Slicer with your GitHub account to obtain a license key:

```bash
slicer activate --help

slicer activate
```

Pro/Commercial users should save their license key (received by email after checking out) to `~/.slicer/LICENSE` without running any additional commands.

Next, start your first VM with the [walk through](/getting-started/walkthrough).

## Snapshot-based storage

Snapshot-based storage enables much faster VM creation times. ZFS is the recommended option for Slicer, devmapper is also supported. See [storage for slicer](/storage/overview) for more info on the different storage backends.

For best performance using a dedicated drive, volume or partition for the storage backend is recommended. If no disk is provided a loopback device will be created automatically.

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
