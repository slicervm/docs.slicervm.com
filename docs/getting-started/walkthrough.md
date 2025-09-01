# Create a Linux VM

In this example we'll walk through how to create a Linux VM using Slicer on an x86_64 host, or an Arm64 host.

The `/dev/kvm` device must exist and be available to continue.

## Create the VM configuration

Slicer is a long lived process that can be run in the foreground or daemonised with systemd.

To create a VM using a Linux bridge, and a disk file for storage create the following `./vm-image.yaml` file:

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

  kernel_image: "ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-latest"
  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker
```

The `storage: image` setting means a disk image will be cloned from the root filesystem into a local file. It's not the fastest option for the initial setup, but it's the simplest, persistent and great for long-living VMs. 

Now, open a new terminal window, or ideally launch `tmux` so you can leave the binary running in the background.

```bash
sudo -E slicer up ./vm-image.yaml
```

Having customised the `github_user` to your own username, your SSH keys will have been fetched from your profile, and preinstalled into the VM.

On your workstation, add any routes that are specified so you can access the VMs on their own network.

Connect with:

```bash
ssh ubuntu@192.168.137.2
```

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

## Configuration for an Arm VM

The only changes needed to the `vm.yaml` file are to the `kernel_image` and `image` fields:

```diff
-  kernel_image: "ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-latest"
-  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"
+  kernel_image: "ghcr.io/openfaasltd/actuated-kernel:6.1.90-aarch64-latest"
+  image: "ghcr.io/openfaasltd/slicer-systemd-arm64:6.1.90-aarch64-latest"
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

The auth token will be created at `/var/lib/slicer/token` and can be used via a `Authorization: Bearer` header.

i.e.

```bash
curl -s http://127.0.0.1:8080/nodes/ -H "Authorization: Bearer $(sudo cat /var/lib/slicer/token)" | jq
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

