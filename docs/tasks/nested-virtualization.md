# Nested Virtualization with Slicer

Nested virtualization refers to running a virtual machine within another.

Generally, you should always aim to run microVMs directly on bare-metal for the best performance and lowest overheads. There are a few use-cases where that is not possible, so nested virtualization provides an alternative at the trade-off of some additional latency.

There are three main use-cases for nested virtualization with Slicer:

1. Your primary OS is MacOS or Windows, which means you have to run Slicer within a Linux VM.
2. You only have access to cloud VMs from DigitalOcean, Azure, or Google Cloud Platform (GCP) rather than bare-metal servers.
3. You want to run Slicer within Slicer for testing and experimentation.

For the first two use-cases, there's nothing extra for you to do. You're simply running all the Slicer commands within an existing VM.

The rest of this guide covers the third use-case: running Slicer within Slicer. This requires attention to IP addresses and routing configurations if you want VMs to be accessible from outside the server.

## Slicer within Slicer

### Topology

This guide covers the following topology. The workstation and server IPs used below are examples, replace them with the actual IPs of your machines:

* **Workstation** (`192.168.1.10`) - Your local machine from which you want to access VMs.
* **Server** (`192.168.1.11`) - A bare-metal server or VM with KVM support running Slicer (e.g., Intel N100, Intel NUC).
* **slicer0** (`192.168.130.0/24`) - The Slicer instance running directly on the server.
* **slicer1** (`192.168.137.0/24`) - A Slicer instance running within a slicer0 VM.

The following diagram shows what runs on the server:

```
┌─────────── Slicer daemon ────────────────┐
│  Host Group: slicer0                     │
│  CIDR: 192.168.130.0/24                  │
│                                          │
│  ┌─────────── Slicer VM ─────────────┐   │
│  │  slicer0-1 (192.168.130.2)        │   │
│  │                                   │   │
│  │  ┌──────── Slicer daemon ──────┐  │   │
│  │  │  Host Group: slicer1        │  │   │
│  │  │  CIDR: 192.168.137.0/24     │  │   │
│  │  │                             │  │   │
│  │  │  ┌─────── Slicer VM ─────┐  │  │   │
│  │  │  │  slicer1-1            │  │  │   │
│  │  │  │  192.168.137.2        │  │  │   │
│  │  │  └───────────────────────┘  │  │   │
│  │  │                             │  │   │
│  │  └─────────────────────────────┘  │   │
│  │                                   │   │
│  └───────────────────────────────────┘   │
│                                          │
└──────────────────────────────────────────┘
```

### Prerequisites

* The slicer0 instance must use a different IP range than slicer1. This example uses `192.168.130.0/24` for slicer0 and `192.168.137.0/24` (the default) for slicer1.
* Use different host group names for each Slicer instance. MAC addresses are derived from the group name, so using the same name could result in duplicate MAC addresses and network conflicts.

### Step 1: Install Slicer on the server

On the server, follow the instructions to [install Slicer](/getting-started/install).

Generate a configuration file with a custom CIDR:

```bash
# Use this if you only want to acces via 'slicer vm shell' and not via SSH
slicer new slicer0 --cidr 192.168.130.0/24 > slicer0.yaml

# Download SSH public keys from a GitHub profile
slicer new slicer0 --cidr 192.168.130.0/24 --github alexellis > slicer0.yaml

# Specify an SSH public key file directly
slicer new slicer0 --cidr 192.168.130.0/24 --ssh-key ~/.ssh/id_ed25519.pub > slicer0.yaml
```

Start Slicer:

```bash
sudo -E slicer up ./slicer0.yaml
```

### Step 2: Configure the slicer0 VM for nested virtualization

Copy the Slicer license from the server into the slicer0 VM:

```bash
sudo -E slicer vm cp ~/.slicer/LICENSE slicer0-1:/home/ubuntu/.slicer/LICENSE
```

Connect to the slicer0 VM using `slicer vm shell`:

```bash
sudo -E slicer vm shell --uid 1000 slicer0-1
```

Enable KVM by loading the kernel modules:

```bash
# For AMD CPUs
sudo modprobe kvm && sudo modprobe kvm_amd

# For Intel CPUs
sudo modprobe kvm && sudo modprobe kvm_intel
```

### Step 3: Install Slicer within the slicer0 VM

Perform an [installation](/getting-started/install.md) of Slicer within the slicer0 VM.

Generate a configuration file for slicer1:

```bash
# Use this if you only want to acces via 'slicer vm shell' and not via SSH
slicer new slicer1 --cidr 192.168.137.0/24 > slicer1.yaml

# Download SSH public keys from a GitHub profile
slicer new slicer1 --cidr 192.168.137.0/24 --github alexellis > slicer1.yaml

# Specify an SSH public key file directly
slicer new slicer1 --cidr 192.168.137.0/24 --ssh-key ~/.ssh/id_ed25519.pub > slicer1.yaml
```

Start Slicer within the slicer0 VM:

```bash
sudo -E slicer up ./slicer1.yaml
```

### Connect to VMs

You can connect to using `slicer vm shell`.

Connect to a slicer0 VM:

```bash
sudo -E slicer vm shell slicer0-1 --uid 1000 
```

Next run the shell command again from with the slicer0 shell to connect the any slicer1 nested VMs:

```bash
sudo -E slicer vm shell slicer1-1 --uid 1000
```

### Routing to nested VMs over the LAN

By adding static routes you can make VMs accessible from any machine on the LAN.

```
┌────────── Workstation ────────────────┐
│  192.168.1.10                         │
└──────────────────┬────────────────────┘
                   │
                   │ LAN
                   │
┌──────────────────┴── Server ─────────────────────────────────────────┐
│  192.168.1.11                                                        │
│                                                                      │
│  ┌─────────── Slicer daemon ──────────────────────────────────────┐  │
│  │  Host Group: slicer0                                           │  │
│  │  CIDR: 192.168.130.0/24                                        │  │
│  │                                                                │  │
│  │  ┌─────────── Slicer VM ───────────────────────────────────┐   │  │
│  │  │  slicer0-1 (192.168.130.2)                              │   │  │
│  │  │                                                         │   │  │
│  │  │  ┌──────── Slicer daemon ───────────────────────────┐   │   │  │
│  │  │  │  Host Group: slicer1                             │   │   │  │
│  │  │  │  CIDR: 192.168.137.0/24                          │   │   │  │
│  │  │  │                                                  │   │   │  │
│  │  │  │  ┌─────── Slicer VM ──────────────────────────┐  │   │   │  │
│  │  │  │  │  slicer1-1 (192.168.137.2)                 │  │   │   │  │
│  │  │  │  └────────────────────────────────────────────┘  │   │   │  │
│  │  │  │                                                  │   │   │  │
│  │  │  └──────────────────────────────────────────────────┘   │   │  │
│  │  │                                                         │   │  │
│  │  └─────────────────────────────────────────────────────────┘   │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Two sets of routes are needed, one on the server and one on the workstation. In the commands below, replace `192.168.1.11` with the IP of your server running Slicer nested VMs.

**On the server**

Add a route so the server knows how to reach the nested slicer1 network through the slicer0 VM:

```bash
# Route to the slicer1 VM network via the slicer0 VM
sudo ip route add 192.168.137.0/24 via 192.168.130.2
```

**On the workstation**

Add routes so the workstation knows to reach both VM networks via the server:

```bash
# Route to the slicer0 VM network via the server
sudo ip route add 192.168.130.0/24 via 192.168.1.11

# Route to the slicer1 VM network via the server
sudo ip route add 192.168.137.0/24 via 192.168.1.11
```

Both workstation routes use the server (`192.168.1.11`) as the gateway. The server then forwards traffic for the slicer1 network to the slicer0 VM (`192.168.130.2`), which forwards it to the nested VMs.

Once routing is configured, you can access VMs directly from your workstation. For example using SSH:

```bash
# Connect to a slicer0 VM
ssh ubuntu@192.168.130.2

# Connect to a slicer1 VM
ssh ubuntu@192.168.137.2
```

Any other network traffic works the same way. For example, if a slicer1 VM runs a web server on port 8080, you can reach it directly from the workstation at `http://192.168.137.2:8080`.
