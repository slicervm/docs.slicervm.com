## VFIO (Virtual Function I/O) Passthrough

VFIO (Virtual Function I/O) is a Linux kernel framework that allows a PCI device to be directly assigned to a virtual machine (VM). This enables the VM to have direct access to the hardware device, providing near-native performance and functionality.

Any device such as a GPU or NIC that is passed through to a VM must first be unbound from its current driver on the host, and is exclusively bound to the VFIO driver.

Slicer supports VFIO passthrough for Nvidia GPUs when using Cloud Hypervisor as the hypervisor as per the example [Ollama with a GPU](/examples/gpu-ollama.md).

Support for NICs for router appliances will be coming shortly.

VFIO is limited to x86_64 systems with hardware support for IOMMU (Intel VT-d or AMD-Vi).

## Enable VFIO on start-up

You must edit the `cmdline` argument of your bootloader to include the following parameters.

Intel:

```
intel_iommu=on iommu=pt
```

AMD:

```
amd_iommu=on iommu=pt
```

If you're using Grub as a bootloader, then edit the `/etc/default/grub` file and add the parameters to the `GRUB_CMDLINE_LINUX_DEFAULT` variable. For example:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt"
``` 

Then update Grub:

```
sudo update-grub
```

Then update your initramfs:

```bash
sudo update-initramfs -c -k $(uname -r)
```

Now reboot your system.

### Optional: Bind devices to VFIO on start-up

We recommend unbinding and rebinding devices to VFIO using the `slicer pci` commands either as and when they're required, or via a systemd unit on start-up.

But, you can also bind devices to VFIO via the `cmdline` argument in your bootloader configuration.

Note: That this method only works at the vendor/device ID level, so if you have multiple GPUs, or multiple NICs of the same time, it's very unlikely that you'll want to bind them all to VFIO because they will not be accessible to the host.

Use `sudo -E slicer pci list` to find the vendor and device IDs of the devices you want to bind to VFIO.

Then add them to the `cmdline` argument as follows, replacing the example IDs below with your own:

```
vfio-pci.ids=10de:2204,10de:1aef
```

## View PCI Devices and their IOMMU Groups

To view the PCI devices and their IOMMU groups, you can use the following command:

```
sudo -E slicer pci list
```

Example output:

```
ADDRESS      CLASS    VENDOR   DEVICE   DESCRIPTION                              IOMMU GROUP  VFIO GROUP   DRIVER
-------      -----    ------   ------   ---------------------------------------- ----------   ----------   ------
0000:0b:00.0 0300     10de     2204     VGA compatible controller: NVIDIA Cor... 28           28           vfio-pci
0000:0b:00.1 0403     10de     1aef     Audio device: NVIDIA Corporation GA10... 28           28           vfio-pci
0000:0c:00.0 0300     10de     2204     VGA compatible controller: NVIDIA Cor... 29           29           vfio-pci
0000:0c:00.1 0403     10de     1aef     Audio device: NVIDIA Corporation GA10... 29           29           vfio-pci
```

## Bind a device to VFIO

You can bind a device to VFIO in two ways:

1. By specifying the device's vendor and device ID in the cmdline arguments for the Kernel.
2. By using the `slicer pci` command to bind the device to VFIO.

We recommend only using 2. because 1. is unable to differentiate between multiple devices of the same type such as two or more NICs or two or more GPUs. You often need at least one of these to be available for the host.

View PCI devices via `sudo -E slicer pci list`.

Then run `sudo -E slicer pci bind <PCI_ADDRESS>` to bind the device to VFIO. Replace `<PCI_ADDRESS>` with the actual PCI address of the device you want to bind.

## Unbind a device from VFIO

To unbind a device from VFIO, you can use the following command:

```
sudo -E slicer pci unbind <PCI_ADDRESS>
```

## Rebind a device to its original driver

Once unbound, you can use `bind` with the `--driver` flag to re-bind it to the original driver on your host system. Or reboot if that's easier.

In the case of the GPU example above, you may want to allocate one of the bound GPUs back to the host for display purposes.

```
sudo -E slicer pci bind 0000:0b:00.0 --driver=nvidia
sudo -E slicer pci bind 0000:0b:00.1 --driver=snd_hda_intel
```

## Troubleshooting

1. Ensure that your CPU and motherboard support IOMMU and that it is enabled in the BIOS/UEFI settings.
2. Also double-check your bootloader i.e. Grub configuration for the command line that's passed to the Linux Kernel. Did you skip `update-grub` or `update-initramfs`?
3. Check the output of `sudo dmesg | grep -e DMAR -e IOMMU` for any errors related to IOMMU initialization.
4. Ensure that the device you are trying to passthrough is not being used by the host system and that it's not already bound to a specific driver.
5. Verify that the device is in its own IOMMU group using `sudo -E slicer pci list` - you typically have to bind every device within an IOMMU group otherwise they cannot be used in a VM.

After checking all of the above, if you find your devices are all mixed into the same IOMMU group, that means your system is not designed for VFIO.

As an alternative, you can deploy/run the so called "[ACS patches](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch))", but they may also have certain security or stability implications. Use at your own risk.

