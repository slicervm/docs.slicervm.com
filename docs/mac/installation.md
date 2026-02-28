# Installation

Slicer for Mac is available on all Slicer license tiers. We've tested on macOS Sequoia and Tahoe. `slicer-mac` does not need `sudo`.

## Install the binaries

You need three binaries: `slicer-mac` (the daemon), `slicer` (the CLI client), and `slicer-tray` (the menu bar app).

If you have the Invididual tier, then first, install the `slicer` CLI and activate your license:

```bash
curl -sLS https://get.slicervm.com | sudo bash
slicer activate
```

The activate command will start an OAuth Device Flow via GitHub. Follow the instructions to complete the flow, and you'll get a license key valid for 30 days. The key is written to `~/.slicer/LICENSE` along with a GitHub token. The token can be used to refresh the license key without going through Device Flow again.

If you have a higher tier than Individual, you'll have received an email with the license key, copy and paste it into a file at `~/.slicer/LICENSE`.

Then use the CLI to install the Mac-specific binaries:

```bash
slicer install slicer-mac ~/slicer-mac
```

During preview, Slicer for Mac does not come with a background service/definition (known as a plist on macOS). So you need to launch it as and when you want it. Either directly in a terminal, or in a `tmux` window.

`tmux` is available via `brew install tmux`, and you can get [brew here](https://brew.sh).

The `slicer` command acts as an API client to Slicer for Mac.

By default, `slicer` auto-discovers the local mac socket at `~/slicer-mac/slicer.sock`, so you usually don't need any socket flags or environment variables for local use.

## Initial configuration.

As a new user, we recommend you do not change the default configuration in any way.

First of all, get used to it, leave it in the path we recommend (`~/slicer-mac`) and explore the use-cases you have in mind.

The `slicer-mac` OCI bundle includes a default `slicer-mac.yaml` in the folder after install, so you can use that file directly.

Only run this if you want to recreate the file because you edited or edited it.

```bash
cd ~/slicer-mac
./slicer-mac new > slicer-mac.yaml
```

The generated `slicer-mac.yaml` has two host groups:

- **`slicer`** (`count: 1`) - your persistent Linux VM, starts with the daemon
- **`sbox`** (`count: 0`) - ephemeral sandboxes, launched on demand for things like coding agents.

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
cd ~/slicer-mac
./slicer-mac up
```

## Start the menu bar app (optional)

The menu bar is an optional extension that gives you quick access to VM status, shells, and controls:

```bash
cd ~/slicer-mac
./slicer-tray
```

![Slicer for Mac menu bar](/images/mac/menu-bar.jpg)

> The Slicer menu bar showing a running VM with options to open a shell, view logs, or shut down.

By default, shells open in Terminal.app. For Ghostty or another terminal:

```bash
slicer-tray --terminal "ghostty"
```

## Set up the Slicer CLI

Verify it's working:

```bash
slicer vm list
HOSTNAME          IP              STATUS     CREATED
--------          --              ------     -------
slicer-1          192.168.64.2    Running    2026-02-10 12:46:14
```

The `slicer-1` VM is your main development environment, and is persistent.

You can shell into it with:

```bash
slicer vm shell slicer-1
```

To launch temporary VMs, run the following to launch a new VM into the `sbox` host group:

```bash
slicer vm launch sbox
```

Then:

```bash
# The following will show the two VMs:
slicer vm list

# Access a shell
slicer vm shell sbox-1

# Copy a file in
slicer vm cp ~/file.txt sbox-1:~/

# Use that file, remotely via exec:
slicer vm exec sbox-1 -- stat ~/file.txt

# Delete that VM
slicer vm delete sbox-1
```

## Run slicer-mac as a background service

```bash
cd ~/slicer-mac

# Simplest, without tray icon / menu:
./slicer-mac install --no-tray

# With tray icon / menu:
./slicer-mac install
```

To uninstall:

```bash
cd ~/slicer-mac
./slicer-mac uninstall
```

## Next steps

- [Linux VM](/mac/your-linux-vm) - mount shared folders, forward Docker and K3s
- [Sandboxes](/mac/launch-sandboxes) - spin up ephemeral VMs
