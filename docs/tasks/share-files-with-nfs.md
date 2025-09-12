# Share files between host and VM with NFS

[NFS](https://wiki.archlinux.org/title/NFS) (Network File System) provides a simple way to share files between your host machine and Slicer VMs. This is particularly useful for development workflows, data processing, or when you need persistent storage that survives VM restarts.

This example shows you how to set up an NFS server on your host machine and mount it from within a Slicer VM.

## Prerequisites

You'll need a running Slicer VM. Follow the [walkthrough](/getting-started/walkthrough) to create one if you haven't already.

## Set up the NFS server on the host

Install the NFS kernel server on your host machine:

```bash
sudo apt update && sudo apt install -y nfs-kernel-server
```

Create a directory to share with your VMs:

```bash
sudo mkdir -p /srv/slicer_share
sudo chown nobody:nogroup /srv/slicer_share
```

Configure the NFS exports by editing `/etc/exports`:

```bash
sudo nano /etc/exports
```

Add the following line to allow your Slicer network access to the share. Replace the network CIDR with your actual Slicer network configuration (the default usded in the walkthrough is `192.168.137.0/24`):

```
/srv/slicer_share 192.168.137.0/24(rw,sync,no_subtree_check)
```

Apply the configuration changes:

```bash
sudo exportfs -ra
```

Start and enable the NFS server:

```bash
sudo systemctl enable --now nfs-server
```

You can verify the export is active:

```bash
sudo exportfs -v
```

## Mount the NFS share in your VM

> This section provides manual setup instructions. For an automated setup using [userdate](/tasks/userdata/), see [Automate NFS setup with userdata](#automate-nfs-setup-with-userdata)

Connect to your VM via SSH:

```bash
ssh ubuntu@192.168.137.2
```

Inside the VM, install the NFS client utilities:

```bash
sudo apt update && sudo apt install -y nfs-common
```

Create a mount point:

```bash
sudo mkdir -p /mnt/slicer_share
```

Mount the NFS share from your host (replace `192.168.137.1` with your actual host IP if different):

```bash
sudo mount -t nfs 192.168.137.1:/srv/slicer_share /mnt/slicer_share
```

Test the connection by creating a file:

```bash
echo "Hello from VM!" | sudo tee /mnt/slicer_share/test.txt
```

You should be able to see this file on your host at `/srv/slicer_share/test.txt`.

## Make the mount persistent

To automatically mount the NFS share when the VM boots, add it to `/etc/fstab`:

```bash
echo "192.168.137.1:/srv/slicer_share /mnt/slicer_share nfs defaults 0 0" | sudo tee -a /etc/fstab
```

Test the fstab entry:

```bash
sudo umount /mnt/slicer_share
sudo mount -a
```

## Automate NFS setup with userdata

You can automate the NFS client setup by adding userdata to your VM configuration.

```yaml
config:
  host_groups:
  - name: vm
    userdata: |
      #!/bin/bash
      # Install NFS client
      apt update && apt install -y nfs-common

      # Create mount point
      mkdir -p /mnt/slicer_share

      # Add to fstab for persistent mounting
      echo "192.168.137.1:/srv/slicer_share /mnt/slicer_share nfs defaults 0 0" >> /etc/fstab

      # Mount immediately
      mount -a
```

With this configuration, you can increase the VM count in your host group and all VMs will automatically have the NFS share mounted.
