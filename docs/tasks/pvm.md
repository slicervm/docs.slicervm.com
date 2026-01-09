# Run slicer without KVM on cloud VMs

PVM (Pagetable Virtual Machine) enables Slicer microVMs to run on cloud VMs without nested virtualization enabled or missing the `/dev/kvm` device.

This is particularly useful for AWS EC2 instances, where bare-metal options are significantly more expensive than regular VMs, or other cloud providers where nested virtualization isn't available.

!!! note "Architecture Support"
    PVM currently supports x86_64 architecture only. Arm64 support is not available.

Running Slicer with PVM requires a custom kernel on the host and compatible guest images. Additionally, a patched Firecracker version is require to support PVM.

## Install PVM kernel on the host

Create a new VM instance on your cloud and select Ubuntu 24.04 as the operating system.

> PVM has been tested on Ubuntu 24.04 on [Amazon EC2](https://aws.amazon.com/ec2) and [Hetzner Cloud](https://www.hetzner.com/cloud).

SSH into the host and install the pre-built PVM kernel packages.

Install arkade if it is not on your system already:

```bash
curl -sLS https://get.arkade.dev | sudo sh
```

Download the kernel packages:

```bash
arkade oci install ghcr.io/openfaasltd/actuated-kernel-pvm-host:x86_64-latest \
  --path .
```

Install the downloaded packages:

```bash
sudo dpkg -i *.deb
```

Configure the kernel boot parameters. PVM requires Page Table Isolation (PTI) to be disabled:

```bash
# Configure GRUB to append pti=off to existing kernel parameters
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/ { s/pti=off//g; s/"$/ pti=off"/; s/  */ /g }' /etc/default/grub
sudo sed -i '/^GRUB_CMDLINE_LINUX=/ { s/pti=off//g; s/"$/ pti=off"/; s/  */ /g }' /etc/default/grub
```

Configure GRUB to use the new kernel:

```bash
# Enable an override to the saved default
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub
sudo update-grub

# View available kernel options
sudo grep -n "menuentry '" /boot/grub/grub.cfg | sed "s/.*menuentry '\(.*\)'.*/\1/"

# Set the new PVM kernel as default
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.12.33"
sudo update-grub

# Update all initramfs images
sudo update-initramfs -u -k all
```

Reboot the system to boot into the PVM kernel:

```bash
sudo reboot
```

After reboot, verify the new kernel is running and load the PVM module:

```bash
# Verify PVM kernel is active
uname -a

# Load the PVM kernel module
sudo modprobe kvm_pvm
```

You can also use `lsmod` to verify the kvm_pvm module is loaded:

```sh
Module                  Size  Used by
kvm_pvm                53248  0
kvm                  1400832  1 kvm_pvm
```

## Install Slicer

Follow the standard [installation instructions for slicer](/getting-started/install/). The installation script automatically downloads a forked Firecracker binary with PVM support.

When creating VMs on a PVM-enabled host, use the PVM-compatible base image.

Update the `image` field in slicer config files:

```yaml
  image: "ghcr.io/openfaasltd/slicer-systemd-2204-pvm:x86_64-latest"
```

Note that at the moment PVM is only supported when using Firecracker as the hypervisor.

Ensure the correct hypervisor is selected in slicer config files:

```yaml
  hypervisor: "firecracker"
```

## Performance considerations

PVM adds virtualization overhead compared to bare-metal KVM. Based on testing:

- I/O intensive workloads may see significant performance impact
- CPU-bound tasks typically see lower overhead
- Long-running services and HTTP workloads are less affected
- Cold start times remain fast

For optimal performance, consider bare-metal hosts with hardware virtualization when possible. PVM is ideal when bare-metal isn't available or cost-effective.
