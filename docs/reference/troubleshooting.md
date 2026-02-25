# Troubleshooting Slicer

For Slicer plans with standard support, Discord is the best first support option. For paid-support plans, email support is available; check your welcome email for details.

## Doesn't boot right or has a networking issue - perhaps GitHub keys aren't being imported?

Look in the log file outputted from slicer. So if you are running i.e. `k3s` as the host group and VM 1 isn't booting:

```bash
sudo cat /var/log/slicer/k3s-1.txt
```

If you can get into the VM via the [SOS console](/reference/sos), then run the following:

```bash
sudo journalctl -u mount-config --no-pager
```

Look for networking issues, or a bad routing.

If your networking equipment is forcing the microVMs to use IPv6, but you do not have IPv6 connectivity, then you could try to disable IPv6.

The host is easier to fix and may mean you can leave the microVMs as they are.

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```

To make it permanent:

Then add the following lines to `/etc/sysctl.conf`:

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Alternatively, you could try disabling only within the microVM on first boot:

```yaml
config:
    host_groups:
    - name: k3s
      userdata: |
        #!/bin/bash
        sysctl -w net.ipv6.conf.all.disable_ipv6=1
        sysctl -w net.ipv6.conf.default.disable_ipv6=1
        sysctl -w net.ipv6.conf.lo.disable_ipv6=1

        echo "net.ipv6.conf.all.disable_ipv6 = 1" |tee -a /etc/sysctl.conf
        echo "net.ipv6.conf.default.disable_ipv6 = 1" |tee -a /etc/sysctl.conf
        echo "net.ipv6.conf.lo.disable_ipv6 = 1" |tee -a /etc/sysctl.conf

        # Cause the SSH keys to re re-imported from GitHub on the next boot
        
        rm -rf /home/ubuntu/.ssh/github_keys_imported

        # Optionally, reboot the VM to re-import the keys from GitHub
        # reboot
```

If you're having issues reaching GitHub for your SSH keys, you can [set them manually](/reference/ssh) in the config or userdata.


## The problem may be fixed by upgrading Slicer

You can upgrade the Slicer binary by running the instructions at the end of the [installation page](/getting-started/install).

## I can't connect over SSH

Have a look at the VM's serial console, written to: `/var/log/slicer/vm-1.txt` where `1` is the VM number, and vm is the host group name. 

## The VM can't connect to the Internet

This can occur when there are old or stable routes in place, for instance, if you added a route for `192.168.137.0/24` on another host, but are now running Slicer on your own workstation.

Check routes with:

```bash
sudo ip route
```

Then delete one with i.e. `sudo ip route del 192.168.137.0/24`.

## I've run out of disk space

There are three places to perform a prune/clean-up.

1. Check there are not a lot of unused `.img` files from various launches of VMs

```bash
sudo find / -name *.img
```

Delete the ones you no longer require. Beware of deleting .img files for VMs that you still need.

2. If you've been working with custom images, prune them from the containerd library:

```bash
sudo ctr -n slicer i ls

sudo ctr -n slicer i ls -q |xargs sudo ctr -n slicer i rm
```

If you are using a snapshotter like ZFS or Devmapper, the above command will delete all images and their snapshots.

So to be more selective, you can delete individual images by name:

For instance:

```bash
sudo ctr -n slicer i rm docker.io/library/ubuntu:22.04
```

3. Remove the */var/run* folder for Slicer

The `/var/run` and `/var/log` folders contain logs, sockets, temporary disks, and extracted Kernel files. This can build up over time. `/var/run` is generally ephemeral, and removed on each reboot.

```bash
sudo rm -rf /var/run/slicer
sudo rm -rf /var/log/slicer
```

For the nuclear option, delete all of the containerd's data, this will remove all images, snapshots, including any from containers that you run via Docker, if that's also installed.

```bash
sudo systemctl stop containerd
sudo rm -rf /var/lib/containerd
sudo systemctl restart containerd
```

If you're using image storage mode and need to resize a VM's disk, see the [storage overview](/storage/overview) for instructions on resizing disk images.
