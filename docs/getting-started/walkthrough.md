# Create a Linux VM

In this example we'll walk through how to create a Linux VM using Slicer on an x86_64 host, or an Arm64 host.

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

**Getting a shell**

You can get an interactive shell with `slicer vm shell` - this is the simplest option and can be run directly on the host, or from another machine over the network if you have the API bound to a TCP address.

When using `bridge` networking, you can run the routing commands printed out on start-up or via `slicer route` on another machine. then you can connect via SSH i.e. `ssh ubuntu@192.168.137.2`.

Do not run the route commands on the host itself because it'll break the networking.

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

Additionally, you can run `slicer vm logs`.

## Managing VMs

Copy a file to the VM:

```bash
slicer vm cp ./file vm-1:~/file
```

Copy a file back to the host:

```bash
slicer vm cp vm-1:~/file ./file
```

Copy a folder to the VM:

```bash
slicer vm cp --mode=tar ~/go/src/github.com/alexellis/arkade vm-1:~/arkade
```

Copy a folder back to the host:

```bash
slicer vm cp --mode=tar vm-1:~/arkade ~/go/src/github.com/alexellis/arkade
```


List running VMs:

```bash
slicer vm list
```

Launch ephemeral VMs:

```bash
slicer vm add
```

Delete a VM:

```bash
slicer vm delete VM_NAME
```

See all available commands with `slicer vm --help`, or refer to the [API reference](/reference/api) for direct HTTP access.
