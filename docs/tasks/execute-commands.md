# Execute commands in a VM

Just like [copying files in and out of a VM](/tasks/copy-files.md), executing commands can be done in several ways:

* Initially, via a userdata script or userdata file specified in the host group or via the API/CLI create command after Slicer has been started.
* Through SSH assuming direct network access is available to the VM via the LAN or a VPN.
* Through Slicer's REST API - using your own client, the SDK or the `slicer` CLI.

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
