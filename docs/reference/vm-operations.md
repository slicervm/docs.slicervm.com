# VM Operations: Copy and Exec

Slicer provides commands to interact with running VMs through the `slicer vm cp` (copy) and `slicer vm exec` (execute) commands.

- Slicer copy allows users to copy a file or folder from the local machine into the vm or vica versa.
- Slicer exec runs a command as a given user or root in the microVM


## Copy files from host to VM

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

## Execute commands

The exec command allows you to run commands remotely on VMs. The first argument is the instance to run the commands on, `--` optionally separates the slicer CLI options from the command to run on the VM.

```bash
# Run a command as root (default)
slicer vm exec vm-1 whoami
```

Slicer allows you to control the command execution context, including user permissions and working directory.

```bash
# Run command as non-root user (UID 1000)
slicer vm exec vm-1 --uid 1000 whoami
```

Change the working directory and run commands with arguments:

```bash
# Execute command in specific directory
slicer vm exec vm-1 --cwd /var/log -- ls -la
```

Control shell interpretation for complex commands:

```bash
# Use default bash shell for complex commands with pipes
slicer vm exec vm-1 -- "ls -l | sort"

# Use custom shell interpreter
slicer vm exec vm-1 --shell /bin/zsh -- "echo $SHELL"

# Execute directly without shell (faster for simple commands)
slicer vm exec vm-1 --shell "" whoami
```

Combine with local commands using pipes and STDIO:

```bash
# Pipe local file content to VM command
cat /etc/hostname | slicer vm exec vm-1 -- base64 --wrap 9999
```

## Guest VM Requirements

Both commands require the `slicer-agent` daemon running within the guest VM. For copy operations, the `tar` utility must be available in the VM's `$PATH`. These requirements are included by default in the various supported Slicer VM images.
