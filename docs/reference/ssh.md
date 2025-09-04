# Log into a VM using SSH

This page covers two concepts:

1. SSH access to a running VM over the network (covered on this page)
2. [Serial Over SSH console (SOS)](/reference/sos.md)

## SSH access to a running VM over the network

Unless you have optimised an image to turn off the bundled OpenSSH server, then it will start when the VM boots.

You can configure VMs within a hostgroup with your SSH keys in two ways.

### Using the `github_user` field

The simplest option is to use GitHub. Set your SSH keys on your profile, then they'll be available at `https://github.com/USER.keys`

```yaml
config:
  github_user: alexellis
```

Only one username can be specified within the `github_user` field.

### Using the `ssh_keys` field

The `ssh_keys` field removes the dependency on GitHub, and speeds up the boot by avoiding a call over the Internet to a remote server.

This method supports multiple keys.

```yaml
config:
  ssh_keys:
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3... user@host
    # For a multi-line key, use YAML's pipe syntax
    - |
        ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB3... user@host
```

### Using the `userdata` field to set up SSH keys

You can also use the `userdata` field to set up SSH keys, as shown in the [Userdata for Slicer VMs](/tasks/userdata) page.

```yaml
config:
    host_groups:
    - name: vm
      userdata: |
        #!/bin/bash
        mkdir -p /home/ubuntu/.ssh
        echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3... user@host" >> /home/ubuntu/.ssh/authorized_keys
        chmod 600 /home/ubuntu/.ssh/authorized_keys
        chown -R ubuntu:ubuntu /home/ubuntu/.ssh
```

If you're running a different OS image such as Rocky Linux, make sure you change the home directory and user/group name accordingly.

The `TARGET_USER` environment variable will also be set within the context of the `userdata` script, so you could use that instead of hard-coding `/home/ubuntu` or `/home/rocky`.
