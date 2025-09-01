# Installation

Don't wait for the perfect system. Slicer can run practically anywhere.

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

A Linux system with KVM is required, so if you see `/dev/kvm`, Slicer will work.

Ubuntu LTS is formally supported, whilst Arch Linux and RHEL-like Operating Systems should work - we won't be able to debug your system.

Ideally, nothing else should be installed on a host that runs Slicer. It should be thought of as a basic appliance - a bare OS, with minimal packages.

## Run these steps

Slicer is evolving, and for the better. Today it uses part of the installation process for [actuated](https://actuated.com).

The installer sets up Firecracker, Cloud Hypervisor, containerd for storage, and a few networking options.

```bash
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

Then, get the slicer binary:

```bash
sudo -E arkade octi install ghcr.io/openfaasltd/slicer:latest --path /usr/local/bin
```

Activate Slicer to obtain a license key:

```bash
slicer activate --help

slicer activate
```

