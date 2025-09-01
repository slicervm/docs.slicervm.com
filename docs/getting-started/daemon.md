# How to run Slicer as a Daemon

The term "daemon" is a well established phrase from UNIX meaning a process that runs in the background. Daemons are usually managed by an init system which can monitor and restart them.

Not only can we monitor Slicer's logs via `journalctl`, but we can manage it with standard `systemctl` commands.

Let's take the example from the [walkthrough](/getting-started/walkthrough) and create a systemd service for it:

Create a service named i.e. `vm-image.service`:

```bash
[Unit]
Description=Slicer

[Service]
User=root
Type=simple
WorkingDirectory=/home/alex
ExecStart=sudo -E /usr/local/bin/slicer up \
  /home/alex/vm-image.yaml \
  --license-file /home/alex/.slicer/LICENSE
Restart=always
RestartSec=30s
KillMode=mixed
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

Install the service, and set it to start up on reboots:

```bash
sudo cp ./vm-image.service /etc/systemd/system/
sudo systemctl enable vm-image.service
```

Now before starting the service, make sure you shut down any existing Slicer process that is managing this particular VM.

Then:

```bash
sudo systemctl start vm-image.service
```

To view the logs for the service run:

```bash
# Page through all logs
sudo journalctl --output=cat -u vm-image

# Tail the latest logs
sudo journalctl --output=cat -f -u vm-image

# View all logs since today/yesterday
sudo journalctl --output=cat --since today -u vm-image
sudo journalctl --output=cat --since yesterday -u vm-image
```

To stop the service run `sudo systemctl stop vm-image`, and to prevent it loading on start-up run: `sudo systemctl disable vm-image`.

You can create multiple Slicer services to run different sets of VMs or configurations on the same host.
