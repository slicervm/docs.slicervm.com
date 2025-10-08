# Installation

Don't wait for the perfect system. Slicer can run practically anywhere.

To activate Slicer, you'll need a Hobbyist or Pro [subscription](https://slicervm.com/pricing) - pick according to your needs. These run month by month, so there's very little commitment or risk involved.

After the installation, when you run `slicer activate` you'll get an invite link to the Discord server. We highly recommend joining.

## System requirements

Any reasonably modern computer can run Slicer, the requirements are very low - x86_64 or Arm64 (including the Raspberry Pi).

Sustainably sourced:

* Low powered N100 mini PC / Beelink / Minisforum
* Ampere Altra / Raspberry Pi 5 (with NVMe)
* Mac Mini M1 / M2 with Asahi Linux installed
* Self-built ATX tower
* Computer under your desk / old Thinkpad / Dell server from eBay

Cloud-based bare-metal:

* Hetzner Robot (cheapest, best value)
* Phoenix NAP
* Latitude.sh

Enterprise:

* On-premises datacenter
* OpenStack / VMware (with nested virtualisation)
* Azure, DigitalOcean, GCP VMs (with nested virtualisation)

A Linux system with KVM is required (bare-metal or nested virtualisation), so if you see `/dev/kvm`, Slicer will work there.

Ubuntu LTS is formally supported, whilst Arch Linux and RHEL-like Operating Systems should work - we won't be able to debug your system.

Ideally, nothing else should be installed on a host that runs Slicer. It should be thought of as a basic appliance - a bare OS, with minimal packages.

## Run these steps

Slicer is evolving, and for the better. Today it uses part of the installation process for [actuated](https://actuated.com).

The installer sets up [Firecracker](https://firecracker-microvm.github.io), [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor), [containerd](https://containerd.io/) for storage, and a few networking options.

```bash
# The installer usually looks for an actuated license, but you don't
# need one to run the installation. We'll create a temporary file via touch.
mkdir -p ~/.actuated
touch ~/.actuated/LICENSE

(
# Install arkade
curl -sLS https://get.arkade.dev | sudo sh

# Use arkade to extract the agent from its OCI container image
arkade oci install ghcr.io/openfaasltd/actuated-agent:latest --path ./agent
chmod +x ./agent/agent*
sudo mv ./agent/agent* /usr/local/bin/
)

(
cd agent
sudo -E ./install.sh
)
```

Then, get the Slicer binary:

```bash
sudo -E arkade oci install ghcr.io/openfaasltd/slicer:latest \
  --path /usr/local/bin
```

The same command can be repeated to update Slicer to future versions.

For Home Edition/Hobbyist users, activate Slicer with your GitHub account to obtain a license key:

```bash
slicer activate --help

slicer activate
```

Pro/Commercial should save their license key (received by email after checking out) to `~/.slicer/LICENSE` without running any additional commands.

Next, start your first VM with the [walk through](/getting-started/walkthrough).

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
