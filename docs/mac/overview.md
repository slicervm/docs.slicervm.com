# Slicer for Mac

Slicer for Mac was built from the ground up to run Linux microVMs on Apple Silicon.

It uses the same familiar API and CLI from Slicer for Linux, using Apple's native [Virtualization framework](https://developer.apple.com/documentation/virtualization) instead of KVM. Only Arm64 hosts and guests are supported, however Rosetta support can be enabled to run Intel/AMD binaries.

Typical use-cases include:

* Real Linux with systemd instead of POSIX compatibility
* Sandboxes for AI coding agents
* Local Kubernetes testing and development
* Exploring software like Hermes Agent or OpenClaw without polluting your Mac

Files can be shared to a microVM via folder sharing using virtiofs, or by `slicer cp` (zero networking copy).

!!! note "macOS versions"
    Slicer for Mac is available on all Slicer license tiers. We've tested on macOS Sequoia and Tahoe. `slicer-mac` does not need `sudo`. Intel Macs are out of scope for Slicer at this time, but you could install Linux on them directly, and use Slicer for Linux instead.

## How it works

Two binaries:

- **`slicer`** - the CLI client (same binary as Slicer for Linux)
- **`slicer-mac`** - the server process aka *daemon* that manages VMs using Apple's Virtualization framework

An [optional menu bar app (`slicer-tray`)](/mac/tray-integration) provides quick access to VM status, shells, and controls.

If you need to run x86_64 binaries, see [Enable Rosetta](/mac/rosetta) after the initial setup.

The config file for Slicer for Mac is named `slicer-mac.yaml`, it defines two host groups. Host groups launch and manage microVMs. The names are flexible on Slicer for Linux, but Slicer for Mac has two fixed groups instead.

- The `slicer` group - a runs a Linux VM that start-up and stays running. This is your day-to-day Linux environment, similar to WSL on Windows, and is fully persistent.
- The `sbox` group aka "sandbox" - is for ephemeral VMs launched on demand through the CLI, API, or by one of your AI coding agents. These VMs are ephemeral by default, but can be made permanent via `slicer vm launch sbox --persistent`

## Conceptual architecture

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

A typical flow, might be to install Docker in `slicer-1`, and then port-forward it back to the host using its UNIX socket, so you can run `docker` directly on your machine.

In a similar way, a sandbox such as `sbox-1` may have K3s installed, exposing port 6443. You can access that by copying the KUBECONFIG file back to your local machine and using the sandbox's IP or the `slicer vm forward` command.

## Networking

Slicer for Mac uses Apple's [VZNATNetworkDeviceAttachment](https://developer.apple.com/documentation/virtualization/vznatnetworkdeviceattachment). Guests access the Internet through your host.

* The host can ingress to guests using their IP or `slicer vm forward`
* Guests can talk to the host to access things like HTTP proxies, or a Docker pull-through cache, etc.
* Unlike Slicer for Linux, which supports bridged mode networking, guests on Slicer for Mac cannot talk to each other, unless you install an overlay network using something like Wireguard, routing through the host.

Firewall rules can be set up using `pf` to prevent guests from accessing certain IP ranges. You can also write your own intercepting proxy, and have `pf` force all NAT traffic through it to enforce your own filtering.

## Storage

Slicer for Mac uses image-backed storage for each virtual machine. The OCI image for the root filesystem and Kernel is unpacked once, then cloned instantly using APFS' native Copy-on-Write support. Launching a new sandbox doesn't copy the entire disk, and only deltas are actually allocated to the host's disk.

If the base image is a `10GB` file, when you launch a new VM, no new bytes will be allocated, until the guest starts to change or add new files.

Slicer for Linux supports ZFS and devmapper for Copy-on-Write support.

## MacBooks and sleep

Host sleep behavior for MacBooks is controlled by `sleep_action` in `slicer-mac.yaml` and affects VM lifecycle.
See [Sleep behavior](/mac/sleep) for full guidance and per-mode behavior.

## Next steps

- [Installation](/mac/installation) - install binaries and start Slicer for Mac
- [Linux VM](/mac/your-linux-vm) - configure your persistent VM, shared folders, Docker, and K3s
- [Sandboxes](/mac/launch-sandboxes) - spin up and tear down ephemeral VMs
- [Coding agents](/mac/coding-agents) - run Claude Code, OpenCode, and other agents inside the VM
