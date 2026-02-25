# Installation

Slicer for Mac is available on all Slicer license tiers. We've tested on macOS Sequoia and Tahoe. `slicer-mac` does not need `sudo`.

## Install the binaries

You need three binaries: `slicer-mac` (the daemon), `slicer` (the CLI client), and `slicer-tray` (the menu bar app).

First, install the `slicer` CLI and activate your license:

```bash
curl -sLS https://get.slicervm.com | sudo bash
slicer activate
```

Then use the CLI to install the Mac-specific assets:

```bash
slicer install slicer-mac ~/slicer-mac
```

## Generate the config

The `slicer-mac` OCI bundle includes a default `slicer-mac.yaml` in the folder after install, so you can use that file directly and skip regeneration.

Only run this if you want to recreate the file:

```bash
slicer-mac new > slicer-mac.yaml
```

The generated `slicer-mac.yaml` has two host groups:

- **`slicer`** (`count: 1`) - your persistent Linux VM, starts with the daemon
- **`sbox`** (`count: 0`) - ephemeral sandboxes, launched on demand

```yaml
config:
  host_groups:
    - name: slicer
      count: 1
      vcpu: 4
      ram_gb: 8
      storage_size: 15G
      share_home: "~/" # Set to "" to disable sharing
      rosetta: true
      network:
        mode: nat
        gateway: 192.168.64.1/24
    - name: sbox
      count: 0
      vcpu: 2
      ram_gb: 4
      storage_size: 15G
      rosetta: true
      network:
        mode: nat
        gateway: 192.168.64.1/24

  image: "ghcr.io/openfaasltd/slicer-systemd-arm64-avz:latest"
  hypervisor: apple

  api:
    socket: "./slicer.sock"
```

The `share_home` field maps your Mac home directory into the VM via VirtioFS. Set it to `""` to disable sharing entirely, or to a sub-path like `"~/code/"` to limit what the VM can see.

Need Rosetta for x86_64 binaries? Follow [Enable Rosetta](/mac/rosetta).

Run the daemon. The first run pulls and prepares the VM image automatically.

## Start the daemon

Start `slicer-mac`:

```bash
slicer-mac up
```

The config file can also be given as an argument:

```bash
slicer-mac up ./slicer-mac.yaml
```

## Start the menu bar app (optional)

The menu bar is an optional extension that gives you quick access to VM status, shells, and controls:

```bash
slicer-tray --url ./slicer.sock
```

![Slicer for Mac menu bar](/images/mac/menu-bar.jpg)

> The Slicer menu bar showing a running VM with options to open a shell, view logs, or shut down.

By default, shells open in Terminal.app. For Ghostty or another terminal:

```bash
slicer-tray --url ./slicer.sock --terminal "ghostty"
```

## Set up the CLI

Rather than passing `--url` on every command, set the `SLICER_URL` environment variable:

```bash
export SLICER_URL=./slicer.sock
```

If you add it to your `~/.zshrc` or `~/.bashrc`, it will be set automatically for every session, but then you must use the whole absolute path to the socket i.e. `~/slicer-mac/slicer.sock` instead of `./slicer.sock`.

Verify it's working:

```bash
slicer vm list
HOSTNAME          IP              STATUS     CREATED
--------          --              ------     -------
slicer-1          192.168.64.2    Running    2026-02-10 12:46:14
```

## Next steps

- [Linux VM](/mac/your-linux-vm) - mount shared folders, forward Docker and K3s
- [Sandboxes](/mac/launch-sandboxes) - spin up ephemeral VMs
