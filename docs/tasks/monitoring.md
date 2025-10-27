# Logs & Monitoring

## Logs

The output of the serial console which contains boot-up messages and output from the init system is available at `/var/log/slicer`.

So for the quickstart, with 1x VM in a hostgroup named vm, you can view the log with:

```bash
sudo tail -f /var/log/slicer/vm-1.txt
```

These logs are also available via the [REST API](/reference/api) or the CLI:

```bash
sudo -E slicer logs vm-1               # Tail the last 20 lines (default)
sudo -E slicer logs vm-1 --lines 50    # Tail the last 50 lines
sudo -E slicer logs vm-1 --lines 0     # Print the whole file
```

## Monitoring

When the `slicer-vmmeter.service` is loaded and running, then system utilization data can be collected via the `slicer vm top` command.

If you need more monitoring that is available, feel free to let us know what you're looking for.

### View utilization via `slicer vm top`

```bash
$ slicer vm top

HOST   IP             CPU  L1    L5    L15   MEM_USED  MEM_FREE  DISK_USED%  DISK_FREE  NET_RX/s  NET_TX/s  DR/s  DW/s  UP
k3s-2  192.168.136.3  2    0.00  0.00  0.00  159MB     3669MB    2.2         29.2GB     0B/s      0B/s      0B/s  0B/s  7m45s
k3s-1  192.168.136.2  2    0.00  0.00  0.00  154MB     3674MB    2.2         29.2GB     0B/s      0B/s      0B/s  0B/s  7m45s
```

Breakdown:

* CPU - vCPU allocated
* L1, L5, L15 - Load average over 1, 5 and 15 minutes
* MEM_USED - Memory used by the VM
* MEM_FREE - Memory free in the VM
* DISK_USED% - Percentage of disk used in the VM
* DISK_FREE - Disk space free in the VM
* NET_RX/s - Network received per second
* NET_TX/s - Network transmitted per second
* DR/s - Disk read per second
* DW/s - Disk write per second
* UP - Uptime of the VM

To simulate resource usage, you could download [Geekbench 6](https://www.geekbench.com/) and run a benchmark. Note that the Arm preview for Geekbench 6 may not fully complete on an Arm system. 

If slicer is running remotely:

```bash
$ slicer vm top --api http://192.168.1.114:8080
```

If slicer is running remotely but requires authentication:

```bash
$ slicer vm top --api http://192.168.1.114:8080 --token TOKEN_VALUE
$ slicer vm top --api http://192.168.1.114:8080 --token-file TOKEN_VALUE
```

When slicer is running locally, and authentication is enabled, use `sudo -E` and slicer will attempt to read the auth token from the default location of `/var/lib/slicer/auth/token`.

```bash
$ sudo slicer vm top
```

### Automated monitoring with `node_exporter`

The Open Source [node_exporter](https://github.com/prometheus/node_exporter) project from the Prometheus project can be used to collect system metrics from each VM, you can install the agent as a systemd unit file through userdata, or a custom base image.

[Prometheus](https://prometheus.io/) or the [Grafana Agent](https://grafana.com/docs/grafana-cloud/agent/) can then be run on the host, or somewhere else to collect the metrics and store them in a time-series database.

