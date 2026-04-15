# Troubleshooting Slicer for Mac

## Check or repair a VM disk image

If your VM crashes, the Mac sleeps at the wrong time, or `slicer-mac` is killed before the guest shuts down cleanly, the VM disk image may need a filesystem check.

Check the filesystem first in read-only mode:

```bash
e2fsck -v -n ./slicer-1.img
```

If errors are found, repair them:

```bash
e2fsck -f ./slicer-1.img
```

You can install `e2fsck` on macOS via Homebrew:

```bash
brew install e2fsprogs
```

## No Internet or LAN access from the VM

One failure mode on macOS is that the VM networking gets stuck on the wrong subnet because the host-side vmnet state becomes stale.

If you also run Docker Desktop, first check that it is using `Docker VMM` mode. That is the supported way to run Docker and Slicer together without subnet conflicts. See [Use Docker from macOS with the Slicer VM socket](/mac/docker).

You may notice one or more of the following:

- `slicer vm list` shows `slicer-1` on `192.168.64.2`, but the VM cannot reach `192.168.64.1`
- the host shows a stale `bridge100` interface on `192.168.65.1`
- DNS and Internet access fail inside the VM even though the Mac itself is online

The symptom is that the VM has no Internet or LAN access.
The cause is that the VM networking is stuck on the wrong subnet.

In this case, the recovery that has worked reliably is:

- rewrite the stale wrong subnet in the vmnet state from `192.168.65` back to `192.168.64`
- clear the DHCP lease database
- remove the generated vmnet state so macOS rebuilds it

### Recovery steps

Stop Slicer for Mac and any other local VM tooling you were using first.

```bash
pkill -f slicer-mac || true
osascript -e 'tell application "Docker" to quit' || true
```

Remove any stale host bridge created for the shared vmnet range:

```bash
sudo ifconfig bridge100 destroy 2>/dev/null || true
```

If the stale vmnet state shows `192.168.65` where you expect `192.168.64`, rewrite it first:

```bash
sudo sed -i '' 's/192\.168\.65/192.168.64/g' \
  /Library/Preferences/SystemConfiguration/com.apple.vmnet.plist
```

Back up and remove the DHCP lease database:

```bash
sudo mv /var/db/dhcpd_leases /var/db/dhcpd_leases.bak.$(date +%s)
```

Back up and remove the generated vmnet state file:

```bash
sudo mv /Library/Preferences/SystemConfiguration/com.apple.vmnet.plist \
  /Library/Preferences/SystemConfiguration/com.apple.vmnet.plist.bak.$(date +%s)
```

Start Slicer for Mac again:

```bash
cd ~/slicer-mac
./slicer-mac up
```

### Verify the fix

Confirm the host bridge and VM gateway were recreated correctly:

```bash
ifconfig -a | rg '^(bridge|vmenet)'
slicer vm list
slicer vm exec slicer-1 -- ping -c 2 192.168.64.1
```

If the VM can reach `192.168.64.1` again, the stale state has been cleared.

### What changed

These are the only host-side changes made by the recovery process:

- removed the stale `bridge100` interface if it existed
- rewrote the stale vmnet subnet from `192.168.65` to `192.168.64`
- removed `/var/db/dhcpd_leases` so stale leases would not be reused
- removed `/Library/Preferences/SystemConfiguration/com.apple.vmnet.plist` so macOS could recreate vmnet state

### Notes

- `bridge0` is usually the macOS Thunderbolt Bridge and is unrelated to this issue
- if you also run Docker Desktop, restart Slicer for Mac first, then confirm networking works before bringing Docker back up
