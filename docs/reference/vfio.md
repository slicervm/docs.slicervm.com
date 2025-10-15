## VFIO (Virtual Function I/O) Passthrough

VFIO (Virtual Function I/O) is a Linux kernel framework that allows a PCI device to be directly assigned to a virtual machine (VM). This enables the VM to have direct access to the hardware device, providing near-native performance and functionality.

Any device such as a GPU or NIC that is passed through to a VM must first be unbound from its current driver on the host, and is exclusively bound to the VFIO driver.

Slicer supports VFIO passthrough for Nvidia GPUs when using Cloud Hypervisor as the hypervisor as per the example [Ollama with a GPU](/examples/gpu-ollama.md).

Support for NICs for router appliances will be coming shortly.

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

Finally, reboot the system to apply the changes.

You can also permanently bind a device to VFIO by adding its vendor and device ID to the `cmdline` argument. For example, to bind an Nvidia GPU - it needs to be bound along with its associated audio device, otherwise it will not work.

```
vfio-pci.ids=10de:2204,10de:1aef
```

Note: if you have two GPUs, it will not be possible to bind only one in this way, so we recommend using the `slicer pci bind` command instead via a systemd unit on start-up.

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

If you see an error such as `Warning: Could not get VFIO information: no vfio device found` then your system may not be compatible with VFIO. Ensure that your CPU and motherboard support IOMMU and that it is enabled in the BIOS/UEFI settings. Also double-check your bootloader i.e. Grub configuration for the command line that's passed to the Linux Kernel.

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

