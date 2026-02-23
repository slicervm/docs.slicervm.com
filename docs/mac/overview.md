# Slicer for Mac

Slicer for Mac runs arm64 Linux VMs on Apple Silicon using Apple's native [Virtualization framework](https://developer.apple.com/documentation/virtualization). It provides a persistent Linux VM with native folder sharing, Docker, K3s, and disposable sandboxes - all driven by the same CLI and REST API as Slicer for Linux.

!!! note "Preview"
    Slicer for Mac is available as a preview for [all subscription levels](https://slicervm.com/pricing). We've tested primarily on macOS Sequoia and Tahoe, with users reporting that Sonoma also works well. `slicer-mac` does not need `sudo`, but `slicer` on Linux generally does for most commands.

## How it works

Two binaries:

- **`slicer`** - the CLI client (same binary as Slicer for Linux) and used to download `slicer-mac`
- **`slicer-mac`** - the daemon that manages VMs using Apple's Virtualization framework

An optional menu bar app (`slicer-tray`) provides quick access to VM status, shells, and controls.
It is shipped as part of the `slicer-mac` OCI asset set.
If you need tray details, see [Tray integration](/mac/tray-integration).
If you need x86_64 support, see [Enable Rosetta](/mac/rosetta).

The daemon reads a `slicer-mac.yaml` config file that defines two host groups:

- **Services** (`slicer` group) - a persistent Linux VM that boots with the daemon and stays running. This is your day-to-day Linux environment, similar to WSL on Windows.
- **Sandboxes** (`sbox` group) - ephemeral VMs launched on demand via the CLI or API. They're destroyed when you restart the daemon, close the lid, or delete them.

# Architecture (conceptual)

```text
                         +----------------------------+
                         |          slicer CLI        |
                         |   (vm shell / vm cp / API) |
                         +-------------+--------------+
                                       |
                                       v
      +--------------------------------+-----------------------------------+
      |              slicer-mac daemon on macOS                            |
      |  Reads `slicer-mac.yaml` and controls local microVMs               |
      +-----------------------+----------------------+---------------------+
                              |                      |
                              |                      |
                              v                      v
             +-----------------------------+    +----------------------------+
             | host_group: slicer          |    | host_group: sbox           |
             | Long-lived primary workload |    | Disposable / on-demand VMs |
             +--------------+--------------+    +-------------+--------------+
                            |                                |
                            v                                v
                     +-------------+                  +----------------+
                     |   slicer-1  |                  |    sbox-1      |
                     | main VM     |                  | sample sbox VM |
                     +-------------+                  +----------------+
```

Docker's socket is port-forwarded to your Mac as a Unix socket, so `docker` commands on the Mac talk directly to the VM. K3s exposes port 6443, so `kubectl` on your Mac can target the cluster running inside `slicer-1`. It feels like you're on Linux.

## Storage

Unlike Slicer for Linux (which supports [devmapper](/storage/devmapper) and [ZFS](/storage/zfs) CoW backends), the Mac version uses image-backed storage. The OCI image is unpacked once, then cloned instantly using APFS' native Copy-on-Write. Launching a new sandbox doesn't copy the entire disk.

## MacBooks and sleep

Host sleep behavior for MacBooks is controlled by `sleep_action` in `slicer-mac.yaml` and affects VM lifecycle.
See [Sleep behavior](/mac/sleep) for full guidance and per-mode behavior.

## Next steps

- [Installation](/mac/installation) - install binaries and start Slicer for Mac
- [Linux VM](/mac/your-linux-vm) - configure your persistent VM, shared folders, Docker, and K3s
- [Sandboxes](/mac/launch-sandboxes) - spin up and tear down ephemeral VMs
- [Coding agents](/mac/coding-agents) - run Claude Code, OpenCode, and other agents inside the VM
