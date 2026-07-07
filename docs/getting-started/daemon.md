# How to run Slicer as a Daemon

The term "daemon" is a well established phrase from UNIX meaning a process that runs in the background. Daemons are usually managed by an init system which can monitor and restart them.

Do you need to run Slicer as a systemd service right off the bat? It depends.

> You may not need a systemd service if you're still in the development and testing phase.
> It's easier and more convenient to start running Slicer in a `tmux`. You can attach and detach as you like, and it'll stay running in the background. A permanent service comes in useful when you need to run Slicer on boot-up and to ensure it gets restarted if it crashes for any reason. Learn how [Alex uses tmux to run processes in the background](https://www.youtube.com/watch?v=kOWlpUiYNlg).

Not only can we monitor Slicer's logs via `journalctl`, but we can manage it with standard `systemctl` commands.

You can run one or many Slicer daemons in this way, just make sure the host group CIDRs do not overlap, and that the API binds to a different UNIX socket or a different TCP port.

## Install Slicer as a systemd service

Let's say you wanted to create a service for a hostgroup named "vm":

```bash
mkdir -p ~/vm/
cd ~/vm/
slicer new vm > ./slicer.yaml
```

To listen on a UNIX port instead, use `slicer new vm --api-auth=false --api-bind ./slicer.sock > ./slicer.yaml`.

Install and start the service:

```bash
sudo slicer service generate --install \
  --name slicer \
  ./slicer.yaml
```

The `--install` flag creates `/etc/systemd/system/slicer.service`, enables it, and starts it. It will not overwrite an existing unit unless you pass `--force`.

By default, the generated unit runs Slicer as `root` and uses the current directory as its working directory unless you pass `--working-directory`.

The license file defaults to the invoking user's `~/.slicer/LICENSE`. If you pass `--user`, Slicer resolves that user's license instead. Pass `--license-file` to use a specific path.

If you need the service process to run as another user, pass `--user`:

```bash
sudo slicer service generate --install \
  --name slicer \
  --user alex \
  ./slicer.yaml
```

When using a non-root service user, the generated unit runs Slicer through `sudo`, so make sure the user has passwordless sudo permission to run Slicer.

Run `visudo`, then add this line to the end of the file, replacing `alex` with your username:

```
alex ALL=(ALL) NOPASSWD: ALL
```

Or you could be more specific and only allow `slicer` to elevate to sudo for your user:

```bash
export USER=alex

echo "$USER ALL=(ALL) NOPASSWD: /usr/local/bin/slicer" | sudo tee /etc/sudoers.d/$USER-slicer
```

Assuming you went for the UNIX socket option, you could run i.e.

```bash
sudo slicer --url ~/vm/slicer.sock vm list
HOSTNAME                  IP              RAM          CPUS     STATUS     CREATED              TAGS                
--------                  --              ---          ----     ------     -------              -----               
vm-1                      192.168.137.2   4GiB         2        Running    2026-02-27 16:30:53      
```

For the TCP option (with the default port of 8080), you'd run:

```bash
sudo slicer vm list
HOSTNAME                  IP              RAM          CPUS     STATUS     CREATED              TAGS                
--------                  --              ---          ----     ------     -------              -----               
vm-1                      192.168.137.2   4GiB         2        Running    2026-02-27 16:30:53      
```

To view the logs for the service run:

```bash
# Page through all logs
sudo journalctl -f --output=cat -u slicer

# Page through all logs
sudo journalctl --output=cat -u slicer

# View all logs since today/yesterday
sudo journalctl --output=cat --since today -u slicer
sudo journalctl --output=cat --since yesterday -u slicer
```

To stop the service run `sudo systemctl stop slicer`, and to prevent it loading on start-up run: `sudo systemctl disable slicer`.

You can create multiple Slicer services to run different sets of VMs or configurations on the same host.

## Preserve VM state across host reboots

If you use the Firecracker hypervisor, you can add `--suspend-on-shutdown` to `slicer service generate` so that Slicer suspends running VMs to disk when the daemon is stopped. On the next daemon start, Slicer automatically restores the VMs that were suspended by the shutdown path.

This is useful for planned host maintenance, such as upgrading the host kernel and rebooting without losing in-memory VM state, long-running shell sessions, or background processes inside the VM.

To enable it, pass `--suspend-on-shutdown` when installing the service:

```bash
sudo slicer service generate --install \
  --name slicer \
  --suspend-on-shutdown \
  ./slicer.yaml
```

The generated unit defaults `TimeoutStopSec` to `120` seconds. Increase it with `--timeout-stop-sec` if your VMs need more time to suspend.

If suspend fails, Slicer falls back to its normal graceful or forceful shutdown behavior.

## Generate a systemd unit

You can also generate the systemd unit without installing it. This is useful if you want to pipe it into another command, save it to a file, or inspect it before installing:

```bash
slicer service generate \
  --name slicer \
  ./slicer.yaml
```

To generate a unit for a non-root service user:

```bash
slicer service generate \
  --name slicer \
  --user alex \
  ./slicer.yaml
```
