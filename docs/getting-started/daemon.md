# How to run Slicer as a Daemon

The term "daemon" is a well established phrase from UNIX meaning a process that runs in the background. Daemons are usually managed by an init system which can monitor and restart them.

Do you need to run Slicer as a systemd service right off the bat? It depends.

> You may not need a systemd service if you're still in the development and testing phase.
> It's easier and more convenient to start running Slicer in a `tmux`. You can attach and detach as you like, and it'll stay running in the background. A permanent service comes in useful when you need to run Slicer on boot-up and to ensure it gets restarted if it crashes for any reason. Learn how [Alex uses tmux to run processes in the background](https://www.youtube.com/watch?v=kOWlpUiYNlg).

Not only can we monitor Slicer's logs via `journalctl`, but we can manage it with standard `systemctl` commands.

You can run one or many Slicer daemons in this way, just make sure the host group CIDRs do not overlap, and that the API binds to a different UNIX socket or a different TCP port.

Let's say you wanted to create a service for a hostgroup named "vm":

```bash
mkdir -p ~/vm/
slicer new vm > ~/vm/slicer.yaml
```

Create a service named i.e. `vm.service`:

```bash
[Unit]
Description=Slicer for vm

[Service]
User=alex
Type=simple
WorkingDirectory=/home/alex/slicer
ExecStart=sudo /usr/local/bin/slicer up \
  /home/alex/slicer/vm/slicer.yaml
Restart=always
RestartSec=30s
KillMode=mixed
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

Install the service, and set it to start up on reboots:

```bash
sudo systemctl enable --now vm.service
```

To view the logs for the service run:

```bash
# Page through all logs
sudo journalctl --output=cat -u vm

# Tail the latest logs
sudo journalctl --output=cat -f -u vm

# View all logs since today/yesterday
sudo journalctl --output=cat --since today -u vm
sudo journalctl --output=cat --since yesterday -u vm
```

To stop the service run `sudo systemctl stop vm`, and to prevent it loading on start-up run: `sudo systemctl disable vm`.

You can create multiple Slicer services to run different sets of VMs or configurations on the same host.
