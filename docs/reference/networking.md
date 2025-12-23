# Networking modes for Slicer microVMs

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
    - name: isolated
  network_name: "slicer"
```

To use CNI, leave off the `network` section of the hostgroup.

The `slicer` network is defined at: `/etc/cni/net.d/51-slicer.conflist` and can be edited. Additional named configs can be created and selected by changing the `network_name` field in the hostgroup config.

## Isolated Mode Networking

Isolated Mode networking is the newest option, and will come in a future release of Slicer.

In this mode, each microVM's TAP is created in a private network namespace, then connected to the host via a veth pair.

Pros:

* Built-in mechanism to block / drop all traffic to specific destinations
* microVMs cannot communicate with each other
* microVMs cannot communicate with the host system
* microVMs cannot communicate with the rest of the LAN

Cons:

* Newer mode, less tested than bridge or CNI modes
* Additional complexity in managing the network namespaces and cleaning up all resources in error conditions or crashes
* Maximum node group name including the suffix `-` and a number, can't be longer than 15 characters. I.e. `agents-1` up to `agents-1000` is fine, but `isolated-agents-1` would not fit.

Example:

```yaml
config:
  host_groups:
    - name: isolated
      network:
        tap_prefix: vmtap
        mode: "isolated"
        range: "169.254.100.0/22"
        drop: ["192.168.1.0/24"]
```

The range of `169.254.100.0/22` gives 127 usable IP address blocks, each containing a /30 subnet. The first IP is for the network, the second is for the gateway, the third is for the microVM, and the fourth is for broadcast.

The `drop` list contains CIDR blocks that should be blocked for all microVMs in this hostgroup. In the example above, all microVMs will have all traffic to the standard LAN network `192.168.1.0/24` dropped before it has a chance to leave the private network namespace.

### Isolated Mode and Netplan

On Ubuntu 22.04 (Server), netplan can take over the veth pair that Slicer creates for the isolated network mode. NetworkManager doesn't tend to have this issue and ships with Ubuntu Desktop.

If you run into issues (confirmed by `ip addr` showing no IP on the `ve-` interfaces), run the following:

```bash
sudo -i # Elevate to a root shell

cat > /etc/systemd/network/00-veth-ignore.network <<EOF
[Match]
Name=ve-* veth*
Driver=veth

[Link]
Unmanaged=yes

[Network]
KeepConfiguration=yes
EOF
```

Edit `/etc/systemd/networkd.conf`, then update the `[Network]` section:

```
[Network]
KeepConfiguration=yes
ManageForeignRoutes=no
```

Then reload and restart Slicer and the isolated network mode microVMs:

```bash
sudo chmod 644 /etc/systemd/network/00-veth-ignore.network
sudo systemctl restart systemd-networkd
```
