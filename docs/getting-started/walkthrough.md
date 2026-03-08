# Create a Linux VM

In this example we'll walk through how to create a Linux VM using Slicer on an x86_64 host, or an Arm64 host.

The `/dev/kvm` device must exist and be available to continue.

## Create the VM configuration

Slicer is a long lived process that can be run in the foreground or as a daemon with systemd.

Config files can be copy and pasted from the docs, or you can generate one from a template by running the below, where `vm` is the name of the hostgroup.

```bash
slicer new vm > vm-image.yaml
```

The default configuration uses a Linux bridge for networking, a disk image for storage, and the Firecracker hypervisor.

```yaml
config:
  host_groups:
  - name: vm
    storage: image
    storage_size: 25G
    count: 1
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24

  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker

  api:
    port: 8080
    bind_address: "127.0.0.1"
```

* `count` - the number of VMs to create - the default is `1` - you can set this to `0` to [create VMs via API instead](/reference/api)
* `vcpu` - the number of virtual CPUs to allocate to each VM
* `ram_gb` - the amount of RAM in GB to allocate to each VM
* `storage` - image is the simplest option to get started
* `storage_size` - for storage backends which support it, you can specify the size of the disk
* `github_user` - your GitHub username, used to fetch your public SSH keys from your profile - additional SSH keys can be added via the [ssh_keys](/reference/ssh) API.

The API can bind to a TCP address (secured with an auth token) or a unix socket (secured by OS filesystem permissions). If you run multiple Slicer instances on the same host, give each one a different port or socket path. See the [API reference](/reference/api) for details.

The `storage: image` setting means a disk image will be cloned from the root filesystem into a local file. It's not the fastest option for the initial setup, but it's the simplest, persistent and great for long-living VMs.

Now, open a new terminal window, or ideally launch `tmux` so you can leave the binary running in the background.

```bash
sudo slicer up ./vm-image.yaml
```

Having customised the `github_user` to your own username, your SSH keys will have been fetched from your profile, and preinstalled into the VM.

On your workstation, add any routes that are specified so you can access the VMs on their own network.

If you need to get the route output back again, you can use the `slicer vm route` command on the host itself, specifying the config file as the argument.

Never run any route commands outputted by Slicer on the host itself. It's not required and will break the networking.

Then, you can connect with SSH:

```bash
ssh ubuntu@192.168.137.2
```

## Quick start with slicer auto

If you want to skip the configuration step, use `slicer auto`:

```bash
sudo slicer auto
```

This will:

* Auto-detect a free bridge network range from `10.0.0.0/8` that doesn't conflict with your existing interfaces
* Pick a random API port (so multiple instances don't clash)
* Import your SSH keys from `~/.ssh/*.pub`
* Write an `auto.yaml` config in the current directory
* Start the API server ready for VMs to be launched

The API port is printed on startup and written to `auto.yaml` for reference.

To launch a VM, open a new terminal and run:

```bash
export SLICER_URL="http://127.0.0.1:PORT"
export SLICER_TOKEN_FILE="/var/lib/slicer/auth/token"

slicer vm add
```

Replace `PORT` with the port shown in the `auto.yaml` file or the startup output.

### Flags

| Flag | Default | Description |
|---|---|---|
| `--count` | `0` | Number of VMs to pre-start (0 means API-only, launch with `slicer vm add`) |
| `--ram` | `4` | RAM in GiB per VM |
| `--cpu` | `2` | vCPUs per VM |
| `--isolated` | `false` | Use isolated networking instead of bridge |
| `--ssh-key` | | SSH public key(s) to add (can be repeated) |
| `--github` | | GitHub username to import SSH keys from |
| `--find-ssh-keys` | `true` | Scan `~/.ssh/` for public keys |
| `--api-port` | auto | Explicit API server port (default: auto-assign free port) |
| `--api-bind` | `127.0.0.1` | API bind address or unix socket path |
| `--socket` | `false` | Use a unix socket at `./auto.sock` instead of TCP |
| `--kernel` | | Path to a custom kernel, or leave blank to extract one from the rootfs |

### Examples

Pre-start VMs with the daemon instead of launching them separately:

```bash
sudo slicer auto --count=1

sudo slicer auto --count=3 --ram=8 --cpu=4
```

SSH keys from `~/.ssh/*.pub` on the host are imported automatically. You can also import keys from a GitHub profile:

```bash
sudo slicer auto --github=alexellis
```

Use isolated networking (no bridge, access VMs via `slicer vm exec` / `slicer vm forward`):

```bash
sudo slicer auto --isolated
```

Use a unix socket instead of TCP for the API:

```bash
sudo slicer auto --socket
```

Set a fixed API port instead of auto-assigning:

```bash
sudo slicer auto --api-port=9090
```

### Re-run behavior

On subsequent runs, `sudo slicer auto` reuses the existing `auto.yaml` - including the same port and CIDR. Pass any flag (like `--count=3`) to regenerate it with new settings.

The generated `auto.yaml` is a standard Slicer config - you can inspect or edit it to learn the format, then graduate to `slicer new` + `slicer up` for full control over networking, storage, and naming.

## Ignore changing SSH host keys

If, like the developers of Slicer, you'll be re-creating many hosts with the same IP addresses, you have two options:

* Memorise and get familiar with the `ssh-keygen -R <ip-address>` command
* Or add the following to your `~/.ssh/config` file to stop it complaining

```
Host 192.168.137.*
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    GlobalKnownHostsFile /dev/null
    CheckHostIP no
    LogLevel QUIET
    User ubuntu
```

Repeat it once for each IP range you use with Slicer.

And bear in mind that you should not do this for production or long-running hosts.

## View the serial console

The logs from the serial console including the output from the boot process are available on disk:

```
$ sudo tail -f /var/log/slicer/vm-1.txt

         Starting OpenBSD Secure Shell server...
[  OK  ] Started OpenBSD Secure Shell server.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.

Ubuntu 22.04.5 LTS vm-1 ttyS0

vm-1 login:
```

If you want to tail the logs from all available VMs at once, use `fstail` via `arkade get fstail`:

```bash
sudo -E fstail /var/log/slicer/
```

## Managing VMs

List running VMs:

```bash
slicer vm list
```

Launch additional VMs without restarting the daemon:

```bash
slicer vm add
```

Delete a VM:

```bash
slicer vm delete VM_NAME
```

See all available commands with `slicer vm --help`, or refer to the [API reference](/reference/api) for direct HTTP access.

