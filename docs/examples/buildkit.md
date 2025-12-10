# Remote Docker builds over SSH

[BuildKit](https://docs.docker.com/build/buildkit/) is a modern backend that powers Docker's build engine. This example shows how to set up a dedicated BuildKit instance in a Slicer VM for isolated, remote container builds.

!!! note
    Whilst Slicer could be used to design a multi-tenant container builder this example is not intended for that use case.

    The setup in this example is intended for use by a developer for building images with a faster remote machine, or a different OS/architecture to their machine.

Using a remote BuildKit instance in a Slicer VM provides several benefits:

- **Isolation**: Build processes are completely isolated from your host system
- **Resource control**: Dedicated CPU and memory resources for builds
- **Security**: Builds run in a sandboxed environment
- **Flexibility**: Easy to scale up resources or create multiple build environments
- **Clean state**: Each VM can be easily reset to a clean state

This example is very minimal and covers the basic setup. You can expand it according to your needs. In the next steps we are going to:

- Install and configure BuildKit automatically on first boot with a userdate script.
- Set up Docker buildx to use the remote BuildKit instance.

## BuildKit installation script

Create the `buildkit.sh` userdata script that will automatically install and configure BuildKit:

```bash
#!/usr/bin/env bash
# BuildKit installation and configuration script
# This script installs buildkitd, configures the buildkit group, and creates a systemd service

#!/usr/bin/env bash
set -euxo pipefail

# Install buildkit
arkade system install buildkitd

# Add a buildkit group
sudo groupadd buildkit

# Add ubuntu user to buildkit group
sudo usermod -aG buildkit ubuntu

# Systemd service for buidlkit (daemonized under systemd)
cat <<'EOF' | sudo tee /etc/systemd/system/buildkitd.service > /dev/null
[Unit]
Description=BuildKit Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/buildkitd --addr unix:///run/buildkit/buildkitd.sock --group buildkit
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now buildkitd
```

## VM configuration

Use `slicer new` to generate a configuration file:

```bash
slicer new buildkit \
  --userdata-file buildkit.sh \
  > buildkit.yaml
```

Use the `--ssh-key` or `--github` flags to add ssh keys so you can connect to the BuildKit instance over SSH. The [Docker buildx remote driver](https://docs.docker.com/build/builders/drivers/remote/) supports connection to a remote BuildKit instance over SSH.

For better build performance, consider increasing the VM resources:

```yaml
    vcpu: 4
    ram_gb: 8
    storage_size: 25G
```

## Start the VM

Start the VM with the following command:

```bash
sudo -E slicer up ./buildkit.yaml
```

## Configure Docker buildx

Once your VM is running and BuildKit is installed, you can configure Docker buildx to use it as a remote builder. For more information about Docker builders, see the [Docker builders documentation](https://docs.docker.com/build/builders/).

### Add VM to known hosts

First, add the Slicer VM to your SSH known hosts to avoid connection issues:

```bash
# Replace with your actual VM IP if different
ssh-keyscan 192.168.137.2 >> ~/.ssh/known_hosts
```

### Create the remote builder

Create a new buildx builder instance that connects to your Slicer VM:

```bash
# Create a new builder named 'slicer' using the remote driver
docker buildx create \
  --name slicer \
  --driver remote \
  ssh://ubuntu@192.168.137.2
```

### Verify the builder

Check that your new builder is available and working:

```bash
# List available builders
docker buildx ls

# Inspect the slicer builder for detailed information
docker buildx inspect slicer
```

When buildx can successfully connect to the builder, the status should show as `running`:

```
slicer          remote
 \_ slicer0      \_ ssh://ubuntu@192.168.137.2    running   v0.24.0    linux/amd64 (+3), linux/386
```

The `inspect` command will show additional details about supported platforms, driver configuration, and connection status.

## Use the remote builder

Once your builder is configured and running, all builds executed with `--builder slicer` will run on the remote BuildKit instance instead of your local machine.

Create a simple Dockerfile:

```dockerfile
FROM alpine:3.19

CMD ["echo", "Hello, BuildKit!"]
```

Build a container using your remote BuildKit instance:

```bash
# Build and tag an image using the remote builder
docker buildx build \
  --builder slicer \
  -t hello-buildkit \
  .
```

The remote builder supports all BuildKit features including multi-platform builds, build secrets, cache mounts, and advanced Dockerfile syntax. Build outputs, logs, and any artifacts are handled seamlessly as if building locally, but with the security and resource isolation benefits of the dedicated VM.

You can also set the remote builder as your default to avoid specifying `--builder` on every command:

```bash
docker buildx use slicer
```

## Troubleshooting

If you're having trouble connecting to the remote builder:

1. **Check VM status**: Ensure the VM is running and BuildKit service is active
   ```bash
   # SSH into the VM and check service status
   ssh ubuntu@192.168.137.2
   sudo systemctl status buildkitd
   ```

2. **Verify SSH connectivity**: Test SSH connection directly
   ```bash
   ssh ubuntu@192.168.137.2 "echo 'SSH connection successful'"
   ```

3. **Check BuildKit socket**: Verify the socket is accessible
   ```bash
   ssh ubuntu@192.168.137.2 "sudo -u ubuntu -g buildkit buildctl --addr unix:///run/buildkit/buildkitd.sock debug info"
   ```

## Further thoughts

This was a basic example to demonstrate how to run BuildKit isolated in a Slicer VM. Some ideas to explore further:

- **TCP connection**: Run BuildKit over TCP with mTLS in the VM to avoid SSH key management
- **Ephemeral VMs**: Use the [Slicer REST API](/reference/api/) to provision temporary VMs for each build, then destroy them.
- **BuildKit pools**: Create pools of BuildKit instances for native multi-platform builds (avoiding QEMU emulation) and shared persistent caching.

The isolated nature of microVMs makes this approach particularly well-suited for enterprise build infrastructure where security, resource control, and clean environments are priorities.
