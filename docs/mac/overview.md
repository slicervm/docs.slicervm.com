# Slicer for Mac

Slicer for Mac was built from the ground up to run Linux microVMs on Apple Silicon.

It uses the same familiar API and CLI from Slicer for Linux, but instead of using KVM, leans heavily on Apple's native [Virtualization framework](https://developer.apple.com/documentation/virtualization).

Typical use-cases include: disposable sandboxes for agents, running local Kubernetes clusters, or getting access to a real Linux system, instead of making do with POSIX compatibility.

Slicer microVMs boot very quickly and have some advanced features like Rosetta for running Intel/AMD binaries, and folder sharing.

!!! note "macOS versions"
    Slicer for Mac is available on all Slicer license tiers. We've tested on macOS Sequoia and Tahoe. `slicer-mac` does not need `sudo`. Intel Macs are out of scope for Slicer at this time, but you could install Linux on them, and use Slicer for Linux instead.

## How it works

Two binaries:

- **`slicer`** - the CLI client (same binary as Slicer for Linux)
- **`slicer-mac`** - the server process aka *daemon* that manages VMs using Apple's Virtualization framework

An [optional menu bar app (`slicer-tray`)](/mac/tray-integration) provides quick access to VM status, shells, and controls.

If you need to run x86_64 binaries, see [Enable Rosetta](/mac/rosetta) after the initial setup.

The config file for Slicer for Mac is named `slicer-mac.yaml`, it defines two host groups. Host groups launch and manage microVMs. The names are flexible on Slicer for Linux, but Slicer for Mac has two fixed groups instead.

- The `slicer` group - a runs a Linux VM that start-up and stays running. This is your day-to-day Linux environment, similar to WSL on Windows, and is fully persistent.
- The ``sbox` group aka "sandbox" - is for ephemeral VMs launched on demand through the CLI, API, or by one of your AI coding agents. They are permanently deleted when you shut them down, shut down Slicer for Mac, or delete them via `slicer vm delete NAME`.

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
