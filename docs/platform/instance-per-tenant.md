# Instance Per Tenant

For stronger isolation, run a separate Slicer daemon per tenant. Each gets its own UNIX socket, its own network range, and its own VM namespace. Tenant A cannot see, manage, or reach tenant B's VMs.

```
┌───────────────────────────────────────────────────────┐
│  Your Application                                     │
│                                                       │
│  Tenant A request ──► /run/slicer/a3cf.sock           │
│  Tenant B request ──► /run/slicer/e7d1.sock           │
└───────────┬───────────────────────────┬───────────────┘
            ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐
│ Slicer (a3cf)       │   │ Slicer (e7d1)       │
│ 169.254.100.0/22    │   │ 169.254.104.0/22    │
│                     │   │                     │
│ ┌────────┐┌────────┐│   │ ┌────────┐┌────────┐│
│ │ a3cf-1 ││ a3cf-2 ││   │ │ e7d1-1 ││ e7d1-2 ││
│ └────────┘└────────┘│   │ └────────┘└────────┘│
└─────────────────────┘   └─────────────────────┘
```

## When to use this

Use a separate instance per tenant when:

* You need API-level isolation - one tenant's requests cannot access another's VMs
* You need network isolation between tenants
* You want independent failure domains - one tenant's daemon crashing does not affect others
* Compliance or security requirements demand full separation

## Configuration

Generate a config per tenant with isolated networking, a UNIX socket, and a non-overlapping IP range. Use [isolated mode networking](/reference/networking/#isolated-mode-networking) so VMs from different tenants cannot communicate.

Tenant A:

```bash
slicer new a3cf \
  --net=isolated \
  --isolated-range 169.254.100.0/22 \
  --socket /run/slicer/a3cf.sock \
  --count=0 \
  --graceful-shutdown=false \
  --drop 192.168.1.0/24 \
  > tenant-a.yaml
```

This produces:

```yaml
config:
  host_groups:
    - name: a3cf
      storage: image
      storage_size: 25G
      count: 0
      vcpu: 2
      ram_gb: 4
      network:
        mode: "isolated"
        range: "169.254.100.0/22"
        drop: ["192.168.1.0/24"]
        allow: ["0.0.0.0/0"]
  image: "ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest"
  hypervisor: firecracker
  graceful_shutdown: false
  api:
    bind_address: "/run/slicer/a3cf.sock"
```

Tenant B:

```bash
slicer new e7d1 \
  --net=isolated \
  --isolated-range 169.254.104.0/22 \
  --socket /run/slicer/e7d1.sock \
  --count=0 \
  --graceful-shutdown=false \
  --drop 192.168.1.0/24 \
  > tenant-b.yaml
```

```yaml
config:
  host_groups:
    - name: e7d1
      storage: image
      storage_size: 25G
      count: 0
      vcpu: 2
      ram_gb: 4
      network:
        mode: "isolated"
        range: "169.254.104.0/22"
        drop: ["192.168.1.0/24"]
        allow: ["0.0.0.0/0"]
  image: "ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest"
  hypervisor: firecracker
  graceful_shutdown: false
  api:
    bind_address: "/run/slicer/e7d1.sock"
```

Each `/22` range provides 256 usable VM slots. Use non-overlapping ranges when running multiple daemons on the same host (e.g. `169.254.100.0/22`, `169.254.104.0/22`, `169.254.108.0/22`).

In isolated mode, each VM gets its own network namespace. VMs cannot communicate with each other, with the host, or with the LAN. The `drop` list blocks specific CIDRs. Auth is disabled by default for UNIX sockets since access is controlled by filesystem permissions.

## Start each daemon

Start each daemon in its own terminal or tmux window:

```bash
sudo slicer up tenant-a.yaml
```

```bash
sudo slicer up tenant-b.yaml
```

Each daemon manages its own VMs. Your application routes requests to the correct socket based on which tenant is making the request.

For production, run each daemon as a systemd service - one unit per tenant. See [running in the background](/getting-started/daemon/) for setup.

## API isolation

Each daemon has its own VM namespace. Listing nodes on tenant A's socket returns only tenant A's VMs:

```bash
TOKEN=$(sudo cat /var/lib/slicer/auth/token)

# Tenant A: create a VM
sudo curl -sf --unix-socket /run/slicer/a3cf.sock \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://localhost/hostgroup/a3cf/nodes \
  -d '{"tags":["user=alice","job=123"]}'

# Tenant B: create a VM
sudo curl -sf --unix-socket /run/slicer/e7d1.sock \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://localhost/hostgroup/e7d1/nodes \
  -d '{"tags":["user=bob","job=456"]}'

# Tenant A sees only its own VMs
sudo curl -sf --unix-socket /run/slicer/a3cf.sock \
  -H "Authorization: Bearer $TOKEN" http://localhost/nodes
```

```json
[{"hostname":"a3cf-1","hostgroup":"a3cf","ip":"169.254.100.2",
  "tags":["user=alice","job=123"],"status":"Running"}]
```

Tenant A cannot manage, exec into, or copy files to tenant B's VMs through its socket.

## See also

* [Single Slicer instance](/platform/single-instance/) - simpler deployment for trusted environments
* [Networking](/reference/networking/) - CIDR configuration and bridge setup
* [REST API reference](/reference/api/) - full endpoint documentation
