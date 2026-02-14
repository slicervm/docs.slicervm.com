# Folder sharing

Slicer for Mac uses VirtioFS to mount your Mac directory into the VM.

The default setup uses `share_home` with `~/`.

## Mount the shared folder

```bash
slicer vm shell slicer-1
```

```bash
sudo mkdir -p /home/ubuntu/host
sudo mount -t virtiofs host /home/ubuntu/host
```

## Make sharing persistent

```bash
echo "host /home/ubuntu/host virtiofs rw,nofail 0 0" | sudo tee -a /etc/fstab
```

## Limit the mount path

```yaml
share_home: "~/code/"
```

Use a sub-path in `slicer-mac.yaml` to avoid exposing your entire home directory.
Restart the daemon after changing `share_home`.
