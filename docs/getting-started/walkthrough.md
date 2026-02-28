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
  ssh:
    port: 2222
    bind_address: "0.0.0.0"
```

* `count` - the number of VMs to create - the default is `1` - you can set this to `0` to [create VMs via API instead](/reference/api)
* `vcpu` - the number of virtual CPUs to allocate to each VM
* `ram_gb` - the amount of RAM in GB to allocate to each VM
* `storage` - image is the simplest option to get started
* `storage_size` - for storage backends which support it, you can specify the size of the disk
* `github_user` - your GitHub username, used to fetch your public SSH keys from your profile - additional SSH keys can be added via the [ssh_keys](/reference/ssh) API.

The HTTP API is enabled by default and can be disabled with `enabled: false`.

The Serial Over SSH (SOS) server is disabled by default and can be enabled by providing a port. Most users will  not need the SOS.

The default `bind_address` for both the API and SSH is `127.0.0.1` - loopback on the host system.

There's no need to provide the HTTP API section unless you plan to run Slicer more than once on the same host, in that case, it's useful to include it so you can change it to a different port on another slicer instance.

**Configuration for an Arm64 Slicer installation**

`slicer new` will generate a configuration file for the correct image for Arm or x86_64, but if you create one manually, you'll need to edit the `image` field.

```diff
-  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"
+  image: "ghcr.io/openfaasltd/slicer-systemd-arm64:6.1.90-aarch64-latest"
```

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

## Launch a second VM

Edit the `count` field, and set it to `2`.

Then hit Control + C and launch slicer again.

You'll see the second VM come online and can connect to it over SSH.

## Enable the HTTP API

The API is used by the `slicer vm` commands, and can also be used directly via `curl`.

```yaml
  api:
    port: 8080
    bind_address: "127.0.0.1:"
    auth:
      enabled: true
```

The auth token will be created at `/var/lib/slicer/auth/token` and can be used via a `Authorization: Bearer` header.

i.e.

```bash
curl -s http://127.0.0.1:8080/nodes -H "Authorization: Bearer $(sudo cat /var/lib/slicer/auth/token)" | jq
```

## Enable the Serial Over SSH Console

The Serial Over SSH (SOS) console can be used to log into a VM without a password, and without any form of networking enabled.

```yaml
  ssh:
    port: 2222
    bind_address: "0.0.0.0:"
```

Example:

```bash
ssh -p 2222 ubuntu@localhost
```
