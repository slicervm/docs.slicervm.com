# Slicer's Serial Over SSH console (SOS)

Slicer's Serial Over SSH (SOS) console allows you manage VMs without having an OpenSSH server running, or an active network connection. SOS works over VSOCK, which is a virtual serial console provided by Firecracker and Cloud Hypervisor.

SSH can be convenient for remote access, but depends on an OpenSSH server being installed and running within the VM. On first boot, an operational OpenSSH server will need to go through a host-keys generation process which can take anywhere from a few hundred milliseconds to a few seconds, if entropy is low.

SSH also depends on a network connection being available, and the system having fully booted.

## Set up SOS in your config file

You'll need to specify the source for authorized SSH keys.

This can be one of either, or both of:

* `github_user` - a GitHub username to fetch public keys from
* `ssh_keys` - a list of public keys to add to the VM

```yaml
config:
  github_user: <your-github-username>
  ssh_keys:
  - <your-ssh-public-key>
  - <another-ssh-public-key>
```

Secondly, you need to bind a specific port and adapter for SOS to listen on.

```yaml
config:
  ssh:
    bind_address: `127.0.0.1:`
    port: 2222
```

To limit connections to only those coming from the Slicer host, use `127.0.0.1:`, but to enable remote SOS access from other hosts use `0.0.0.0:`.

The port number can be anything, but it's suggested that you use a higher port i.e. `2221` or `2222`, etc to avoid conflicts.

Each Slicer process can only bind to one port, so if you want to run multiple Slicer instances on the same host, you'll need to use different ports.

## Connect to a VM using SOS

When connecting to the SOS, you'll be presented with a menu.

This menu lists all running VMs, hit up or down arrows and then "Enter" on the target VM.

```bash
Select a VM (use ↑↓ arrows, Enter to select):
----------------------------------------
> vm-1 <
  Quit
----------------------------------------
Press Enter to select, q to quit

```

From within the VM sub-menu, you can:

* Connect - launch a shell like an SSH session as the root user, but without any need for OpenSSH
* Pause the VM - with Firecracker microVMs, the VM's vCPUs are stopped, but memory is still allocated
* Resume the VM - if the VM was paused, this will allow it to continue running from where it left off
* Shutdown the VM - this is a graceful shutdown, equivalent to running `sudo shutdown now` within the VM

```bash
Actions for VM: vm-1
----------------------------------------
> Connect <
  Pause
  Shutdown
  Back
----------------------------------------
Press Enter to select, Esc or 'b' to go back

```

When you connect, you'll be logged in as the `root` user.

```bash
Connecting to: vm-1
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.10.240 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
root@vm-1:/root# 
```

Type `exit` or press `Ctrl+D` to exit the shell session.

To enter the menu again, reconnect with the SSH command you used earlier.

