# Installation

Slicer for Mac is included in the [Home Edition](https://slicervm.com/pricing) and commercial licenses. You'll need macOS Sequoia or later on Apple Silicon.

## Install the binaries

You need three binaries: `slicer-mac` (the daemon), `slicer` (the CLI client), and `slicer-tray` (the menu bar app).

Install Arkade, then use it to install from OCI:

```bash
curl -sLS https://get.arkade.dev | sudo bash
```

Change to the destination directory and install all mac assets:

```bash
mkdir -p ~/slicer-mac
cd ~/slicer-mac
arkade oci install docker.io/alexellis2/slicer-mac:latest .
```

You can also install the CLI from the Linux package to match your license:

```bash
curl -sLS https://get.slicervm.com | sudo bash
slicer activate
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

Start `slicer-mac` with the config file:

```bash
slicer-mac --config ./slicer-mac.yaml
```

## Start the menu bar app (optional)

The menu bar app gives you quick access to VM status, shells, and controls:

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

Add that to your `~/.zshrc` or `~/.bashrc` to make it permanent. Verify it's working:

```bash
slicer vm list
HOSTNAME          IP              STATUS     CREATED
--------          --              ------     -------
slicer-1          192.168.64.2    Running    2026-02-10 12:46:14
```

## Next steps

- [Linux VM](/mac/your-linux-vm) - mount shared folders, forward Docker and K3s
- [Sandboxes](/mac/launch-sandboxes) - spin up ephemeral VMs
