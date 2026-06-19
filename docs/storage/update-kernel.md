# Update the kernel for a persistent VM

When you change the image of a persistent VM to a newer Kernel version, for example moving from a `5.x` Kernel to a `6.x` Kernel, the VM disk still contains the Kernel modules, boot config, and `vmlinux` from the *original* image.

Anything that loads a Kernel module dynamically (Docker, Kubernetes, ZFS, etc.) will then break, because the modules on disk no longer match the running Kernel.

The `slicer disk update-kernel` command provides a migration path: it copies the Kernel modules and boot files from a newer Slicer image into the persistent disk of a stopped VM, so you don't have to recreate the disk from scratch.

This works for all storage modes: [disk images](/storage/overview/#disk-images), [ZFS](/storage/zfs), and [devmapper](/storage/devmapper).

## Performing the upgrade

The upgrade copies the Kernel modules and boot files from a newer Slicer image into the VM's persistent disk, then runs `depmod` so the modules match the new Kernel.

The VMs must be shut down first. All persistent VMs belonging to a Slicer daemon share a single `image` in the configuration, so they must all be upgraded together in the same pass.

Running the `slicer disk update-kernel` command for a VM will:

1. Pull/unpack the new Slicer image and mount its snapshot read-only
2. Mount the target VM disk read-write
3. Copy `lib/modules/<kver>`, `boot/config-<kver>`, and the Kernel binary (`boot/vmlinux`, or `boot/Image` on arm64) into the VM disk
4. Run `depmod` for the new Kernel version against the VM disk

That command handles a single VM's disk. To upgrade a whole host group, stop the VMs, run the command for each one, then bring them back up on the new image:

**1. Stop the Slicer daemon**

Shut down the Slicer daemon, whether you run it as a systemd service, in a tmux session or in the foreground. Stopping the daemon shuts down all running VMs in the host group, releasing their disks so they can be updated.

**2. Select the new image**

Pick the image you want to upgrade to from the [list of image tags](/reference/images).

Tags such as `...-latest` are mutable and move over time. For a reproducible upgrade, it is recommended to pin the image to an exact `@sha256:...` digest for the upgrade command below and in the Slicer config file:

```bash
# Resolve the latest tag to an immutable digest
crane digest ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest
# sha256:1a2b3c...

# Use the pinned reference for the upgrade
ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest@sha256:1a2b3c...
```

**3. Run the upgrade for each VM in the host group**

Run `slicer disk update-kernel` for each VM in the host group, using the image selected in the previous step. The `VM_NAME` argument is the VM's hostname.

Image storage:

The VM name matches the `.img` filename on disk (e.g. `vm-1.img` → `vm-1`). By default the command looks for `<VM_NAME>.img` in the current directory; use `--disk` to point to another location.

```bash
# Looks for ./vm-1.img
sudo slicer disk update-kernel vm-1 \
  ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest@sha256:1a2b3c...

# Explicit .img path
sudo slicer disk update-kernel vm-1 \
  ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest@sha256:1a2b3c... \
  --disk /var/lib/slicer/vm-1.img
```

ZFS or devmapper storage:

List the disks offline with `slicer disk list`. The lease ID contains the VM name (e.g. `slicer/vm-1` → `vm-1`).

```bash
sudo slicer disk update-kernel vm-1 \
  ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest@sha256:1a2b3c... \
  --storage zfs

sudo slicer disk update-kernel vm-1 \
  ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest@sha256:1a2b3c... \
  --storage devmapper
```

**4. Update the configuration and start the daemon**

Update the `image` field in your Slicer configuration to the same image reference you used for the upgrade.

Slicer pins the resolved image digest in a lock file next to your config (e.g. `slicer.yaml.lock`). This lock takes precedence over the `image` field, so it must be removed for the new image to take effect:

```bash
rm -f ./slicer.yaml.lock
```

Then start the daemon again, using the same method you used to run it before.

On boot, each VM will find the matching modules on its disk. Confirm the new Kernel is running with:

```bash
slicer vm exec vm-1 -- "uname -r"
```

!!! note "Out-of-tree modules must be rebuilt"
    Only the Kernel modules shipped in the base image are migrated. Any module that was installed or built separately inside the VM (out-of-tree modules such as DKMS drivers) will not match the new Kernel and must be rebuilt or reinstalled against it after the upgrade.
