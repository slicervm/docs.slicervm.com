# YAML Configuration Reference

Slicer uses a YAML configuration file to define host groups, networking, storage, and API settings. Generate one with `slicer new` or let `slicer auto` create one for you.

## Generate a config file

```bash
slicer new vm > config.yaml
```

The argument (`vm`) becomes the host group name. `slicer new` detects the host architecture and selects the correct image automatically.

## Full example

```yaml
config:
  host_groups:
  - name: vm
    storage: image
    storage_size: 25G
    count: 1
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24

  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker

  api:
    port: 8080
    bind_address: "127.0.0.1"
```

## Host group fields

Each entry in `host_groups` defines a group of identically-configured VMs.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | | Host group name, used to derive bridge and tap names |
| `count` | int | `1` | Number of VMs to create. Set to `0` to launch VMs via API instead |
| `vcpu` | int | `2` | vCPUs per VM |
| `ram_gb` | int | `4` | RAM in GiB per VM |
| `ram_mb` | int | | RAM in MiB (alternative to `ram_gb`) |
| `ram_bytes` | int | | RAM in bytes (alternative to `ram_gb`) |
| `storage` | string | `devmapper` | Storage backend: `image`, `devmapper`, or `zfs` |
| `storage_size` | string | | Disk size, e.g. `25G` or `512M`. Required for `image` storage |
| `persistent` | bool | `false` | Keep root filesystem after VM shutdown |
| `userdata` | string | | Inline cloud-init user data |
| `userdata_file` | string | | Path to a cloud-init user data file |
| `dns_servers` | list | `["8.8.8.8", "1.1.1.1"]` | DNS servers for VMs |
| `gpu_count` | int | `0` | Number of GPUs to pass through |

Only one of `ram_gb`, `ram_mb`, or `ram_bytes` can be specified.

## Networking

See the [networking reference](/reference/networking) for detailed configuration.

### Bridge mode (default)

VMs get routable IPs on a Linux bridge and are reachable via SSH from the host.

| Field | Type | Description |
|---|---|---|
| `network.bridge` | string | Bridge interface name, e.g. `brvm0` |
| `network.tap_prefix` | string | Prefix for TAP interfaces (max 14 chars) |
| `network.gateway` | string | Gateway in CIDR notation, e.g. `192.168.137.1/24` |
| `network.addresses` | list | Optional static IPs. Auto-assigned from the gateway range if empty |

### Isolated mode

VMs have no inbound access from the host network. Use `slicer vm exec`, `slicer vm forward`, or `slicer vm shell` to interact with them.

| Field | Type | Description |
|---|---|---|
| `network.mode` | string | Set to `isolated` |
| `network.drop` | list | CIDRs to block |
| `network.allow` | list | CIDRs to allow (whitelist mode if no `drop` rules) |

## Storage

| Backend | Notes |
|---|---|
| `image` | Disk image cloned from the rootfs. Simplest to set up. Requires `storage_size`. See the [walkthrough](/getting-started/walkthrough). |
| `devmapper` | Device mapper snapshots. Fastest cloning. See [devmapper docs](/storage/devmapper). |
| `zfs` | ZFS volumes. See [ZFS docs](/storage/zfs). |

## SSH access

| Field | Type | Description |
|---|---|---|
| `github_user` | string | GitHub username - public SSH keys are fetched from the profile |
| `ssh_keys` | list | SSH public keys as strings |

At least one of `github_user` or `ssh_keys` is needed for SSH access. Keys can also be managed via the [SSH keys API](/reference/ssh).

## API

The API can bind to a TCP address or a unix socket.

With TCP, requests are authenticated with a bearer token written to `/var/lib/slicer/auth/token`. With a unix socket, the OS filesystem permissions control access so no token is needed.

| Field | Type | Default | Description |
|---|---|---|---|
| `api.port` | int | `8080` | TCP port |
| `api.bind_address` | string | `127.0.0.1` | TCP address or unix socket path (e.g. `./slicer.sock`) |
| `api.auth.enabled` | bool | `true` | Enable bearer token authentication |

See the [API reference](/reference/api) for endpoint documentation.

## Other fields

| Field | Type | Default | Description |
|---|---|---|---|
| `image` | string | | OCI image reference for the root filesystem |
| `kernel_image` | string | | OCI image containing a kernel (optional) |
| `kernel_file` | string | | Path to a kernel binary (optional, extracted from rootfs if not set) |
| `hypervisor` | string | `firecracker` | `firecracker` or `cloud-hypervisor` |
| `graceful_shutdown` | bool | `true` | Send ACPI shutdown before killing VMs |
| `pci` | map | | PCI device passthrough. See [VFIO docs](/reference/vfio) |
