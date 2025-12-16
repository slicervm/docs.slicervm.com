# Copy files to and from a VM

You can copy files to and from a VM using the following methods:

* SSH - sftp, scp, rsync, restic
* Slicer's native mechanism - `slicer vm cp` powered by the `slicer-agent` service

The first approach is fairly standard, but requires SSH to be started, and installed. Any client must have a route and direct access to the VM via the LAN or over some form of VPN/overlay network.

The second approach can be run from anywhere, so long as the client has access to Slicer's REST API, making it more powerful and easier to use.

## Copy files or directories between a client and a VM

The copy command enables bidirectional file transfer between the host and guest VMs without the need of mounting any folders to VMs.

For example, to copy a file named `local-file.txt` from the host to a VM named `vm-1` use the command:

```bash
slicer vm cp example.txt vm-1:.
```

To copy a file from the instance named `vm-1` run:

```bash
slicer vm cp vm-1:/home/ubuntu/example.txt .
```

It is also possible to copy an entire directory between host and a VM:

```bash
# Copy a directory from local machine to VM
slicer vm cp /etc vm-1:/tmp/etc

# Copy a directory from VM to local machine
slicer vm cp vm-1:/etc /tmp/etc
```

Slicer supports two copy modes. The default tar mode supports directories and preserves file structure using compression for efficient transfer. Binary mode provides file-by-file transfer and supports custom permissions setting, making it ideal for single files with specific permission requirements:

```bash
# Use tar mode explicitly (default)
slicer vm cp --mode tar vm-1:/home/ubuntu/documents ./documents

# Use binary mode with custom permissions
slicer vm cp --mode binary --permissions 0755 ./script.sh vm-1:/usr/local/bin/script.sh
```
