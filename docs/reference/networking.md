# Networking modes for Slicer microVMs

Table of contents:

* [Bridge Networking](#bridge-networking)
* [CNI Networking](#cni-networking)
* [Isolated Mode Networking](#isolated-mode-networking)
  * [Quick start](#quick-start)
  * [Specifying an explicit range](#specifying-an-explicit-range)
  * [Drop and allow rules](#drop-and-allow-rules)
  * [Firewall](#firewall)
  * [Additional configuration for Netplan](#additional-configuration-for-netplan)

Both Firecracker and Cloud Hypervisor microVMs use TAP devices for networking. A Linux TAP operates at Layer 2 (Data Link Layer), one end is attached to the microVM, and the other end is attached to the host system.

Slicer supports multiple networking modes for microVMs, each with their own pros and cons. Generally, you should use the default option (Bridge Networking) unless you have specific requirements that need a different mode.

See `slicer new --help` to generate a Slicer config with bridge-based networking. You can also provide a CIDR block to allocate IP addresses from, if you need to run Slicer across different machines and have them all be routable on the local network.

## Bridge Networking

The default for Slicer is to use create a Linux bridge per Slicer daemon, and to attach all microVMs within that hostgroup to the bridge. This allows microVMs to communicate with each other and with the host system.

The bridge also makes the microVMs routable from outside the host system on the local network, subject to the other machine having direct access to the machine where the Slicer daemon is running, and adding a route to the microVM subnet.

Pros:

* Default mode, and best tested option
* Easy to route to microVMs from outside the host system
* Allocate a fixed set of IP addresses, or allocate from a larger subnet

Cons:

* microVMs can communicate with the host, and the rest of the LAN, which may not be desirable in some use-cases
* There can be learning delays with a Linux bridge, so occasionally a microVM may boot up and not be able to resolve DNS for a few seconds

Example with IPs allocated from the subnet automatically:

```yaml
config:
  host_groups:
  - name: vm
    count: 3
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24
```

To assign from a static list of IP addresses, use the `addresses` field, and make sure the `gateway` is in the same subnet:

```yaml
config:
  host_groups:
  - name: vm
    count: 3
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24
      addresses:
      - 192.168.137.2/24
      - 192.168.137.3/24
      - 192.168.137.4/24
```

## CNI Networking

Container Network Interface (CNI) is a popular networking standard in the container ecosystem, and is used by OpenFaaS Edge (faasd), Kubernetes, and various other low-level projects.

Slicer ships with a CNI configuration that acts in a similar way to the bridge mode. Just like with the bridge mode, microVMs can communicate with each other and with the host system, and are routable from outside the host system on the local network.

The combination of `bridge`, and `firewall` CNI plugins are used in the default configuration, however you can also change this as needed for instance to bypass the `bridge` plugin and use something simpler like `ptp`.

Pros:

* Network addresses are allocated from a large, fixed pool defined in a config file

Cons:

* Reliance on CNI to manage networking
* microVMs can communicate with the host, and the rest of the LAN, which may not be desirable in some use-cases
* May also have learning delays like the bridge mode


Example:

```yaml
config:
  host_groups:
    - name: vm
  network_name: "slicer"
```

To use CNI, leave off the `network` section of the hostgroup.

The `slicer` network is defined at: `/etc/cni/net.d/51-slicer.conflist` and can be edited. Additional named configs can be created and selected by changing the `network_name` field in the hostgroup config.

## Isolated Mode Networking

In this mode, each microVM's TAP is created in a private network namespace, then connected to the host via a veth pair. Each VM is fully isolated from the others, from the host, and from the LAN.

Pros:

* Built-in mechanism to block / drop all traffic to specific destinations
* microVMs cannot communicate with each other
* microVMs cannot communicate with the host system
* microVMs cannot communicate with the rest of the LAN

Cons:

* Newer mode, less tested than bridge or CNI modes
* Additional complexity in managing the network namespaces and cleaning up all resources in error conditions or crashes
* Maximum node group name including the suffix `-` and a number, can't be longer than 15 characters. I.e. `agents-1` up to `agents-1000` is fine, but `isolated-agents-1` would not fit.

### Quick start

The easiest way to use isolated mode is with `slicer new` — no IP range is needed:

```bash
slicer new sandbox --net=isolated
```

This generates a minimal config with no `range:` field. Slicer automatically derives a `/22` subnet within `169.254.0.0/16` from the hostgroup name. Each `/22` provides 256 usable `/30` blocks (one per VM).

To add firewall rules at generation time:

```bash
slicer new sandbox --net=isolated --drop 192.168.1.0/24
```

The generated YAML looks like:

```yaml
config:
  host_groups:
    - name: sandbox
      network:
        mode: "isolated"
        drop: ["192.168.1.0/24"]
```

This is safe to use with multiple Slicer daemons running on the same host — each hostgroup name hashes to a different subnet, and Slicer checks for IP collisions on host interfaces before assigning each `/30` block, skipping any that are already in use.

### Specifying an explicit range

If you prefer to control the exact subnet, you can specify a `range:` field. This is useful when you want deterministic IP assignments or need to coordinate ranges across many host groups manually.

```yaml
config:
  host_groups:
    - name: sandbox
      network:
        mode: "isolated"
        range: "169.254.100.0/22"
        drop: ["192.168.1.0/24"]
```

A `/22` range gives 256 usable `/30` blocks. Each `/30` contains 4 IPs: network, gateway, host (microVM), and broadcast.

When using explicit ranges with multiple host groups or multiple Slicer daemons on the same host, use non-overlapping subnets:

* `169.254.100.0/22`
* `169.254.104.0/22`
* `169.254.108.0/22`
* `169.254.112.0/22`

When Slicer is running on different hosts, you can re-use the same subnet ranges on different machines.

You can also pass an explicit range via `slicer new`:

```bash
slicer new sandbox --net=isolated --isolated-range 169.254.100.0/22
```

### Drop and allow rules

The `drop` list contains CIDR blocks that should be blocked for all microVMs in this hostgroup. In the example above, all microVMs will have all traffic to the standard LAN network `192.168.1.0/24` dropped before it has a chance to leave the private network namespace.

### Firewall

There is both a `drop` and an `allow` list that can be given in the networking section.

When only `drop` is given, all other traffic is allowed which hasn't been explicitly blocked.

When only `allow` is given, all other traffic is blocked which hasn't been explicitly allowed.

When neither `drop`, nor `allow` are given, then all traffic is allowed.

### Additional configuration for Netplan

On Ubuntu 22.04 (Server), netplan can take over the veth pair that Slicer creates for the isolated network mode. NetworkManager doesn't tend to have this issue and ships with Ubuntu Desktop.

If you run into issues (confirmed by `ip addr` showing no IP on the `ve-` interfaces), run the following:

```bash
cat <<EOF | sudo tee /etc/systemd/network/00-veth-ignore.network > /dev/null
[Match]
Name=ve-* veth*
Driver=veth

[Link]
Unmanaged=yes

[Network]
KeepConfiguration=yes
EOF
```

Update `/etc/systemd/networkd.conf` as per the following:

```bash
cat <<EOF | sudo tee /etc/systemd/networkd.conf > /dev/null
[Network]
KeepConfiguration=yes
ManageForeignRoutes=no
#SpeedMeter=no
#SpeedMeterIntervalSec=10sec
#ManageForeignRoutingPolicyRules=yes
#ManageForeignRoutes=yes
#RouteTable=
#IPv6PrivacyExtensions=no

[DHCPv4]
#DUIDType=vendor
#DUIDRawData=

[DHCPv6]
#DUIDType=vendor
#DUIDRawData=
EOF
```

Then reload and restart Slicer and the isolated network mode microVMs:

```bash
sudo chmod 644 /etc/systemd/network/00-veth-ignore.network
sudo systemctl restart systemd-networkd
```
