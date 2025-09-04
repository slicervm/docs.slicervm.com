# Troubleshooting Slicer

The best place to get help with the Home Edition is via Discord. For Pro users, email support is available, check the welcome email for details.

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

