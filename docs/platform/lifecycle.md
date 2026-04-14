# VM Lifecycle

Two things shape how you integrate with Slicer: how VMs are named, and how long they live. Both follow from a single primitive, the **host group**, defined in the daemon's config file.

## Host groups

A host group is a named pool of VMs that share the same hardware profile (vCPU, RAM, storage), network, and image. The daemon reads its host groups from its YAML config (typically produced by `slicer new`) at startup. You can define multiple groups in a single config. The common pattern is one group for a long-lived control plane plus one for ephemeral tenant or workload VMs:

```yaml
config:
  host_groups:
    # One persistent, always-on VM created at daemon start.
    - name: ctrl
      storage: image
      storage_size: 25G
      count: 1
      vcpu: 4
      ram_gb: 8
      userdata: |
        apt-get update -qy
        apt-get install -qy nginx postgresql
      network:
        bridge: brctrl0
        tap_prefix: ctrl
        gateway: 192.168.137.1/24

    # No pre-allocated VMs, everything is API-launched.
    # Isolated networking, firewalled subnet with egress controlled by allow/drop.
    - name: sbox
      storage: image
      storage_size: 25G
      count: 0
      vcpu: 2
      ram_gb: 4
      network:
        mode: "isolated"
        drop: []
        allow: ["0.0.0.0/0"]

  image: "ghcr.io/openfaasltd/slicer-systemd-min:6.1.90-x86_64-latest"
  hypervisor: firecracker
  api:
    port: 8080
    bind_address: "127.0.0.1"
    auth:
      enabled: true
```

Run `slicer new --help` for the full set of flags and `slicer new NAME > slicer.yaml` to generate a starter config.

Host groups must be defined in the YAML file before starting the daemon. They cannot be added dynamically at runtime. If you have that need, consider the [Slicer per tenant](/platform/instance-per-tenant/) model, where each tenant gets its own daemon and its own host group config.

### What `count:` does

- **`count: N`**: Slicer creates and **protects** N VMs in that group at startup. They're persistent by construction: the daemon restores them after its own restart, and they come back automatically if the host reboots. Use this for the control plane, a shared database, or anything that must be present whenever the daemon is running.
- **`count: 0`**: no pre-allocated VMs. Callers create and delete VMs on demand through the API (`POST /hostgroup/NAME/nodes`). This is the right shape for sandboxes, per-job workers, and tenant workloads.

Both shapes can coexist in the same daemon. The split above, one persistent control-plane host group plus one on-demand sandbox host group, is how most multi-tenant deployments are structured.

### VM size at launch

The host group's `vcpu` / `ram_gb` are the **default and maximum** for VMs in that group. When you launch a VM through the API you can:

- Omit `cpus` / `ram_bytes` entirely - the VM gets the host group's defaults.
- Request **the same or less** than the defaults - honoured as-is.
- Request **more** than the defaults - rejected with `400`.

So if `sbox` is defined at 2 vCPU / 4 GiB, a client can legitimately launch a 1 vCPU / 1 GiB worker inside it but cannot launch an 8 vCPU / 16 GiB worker. To offer bigger VMs, define a separate host group with a bigger profile.

## Naming: pets vs. cattle

You don't pick the hostname. When Slicer creates a VM in a host group, it assigns the name: `<hostgroup>-1`, `<hostgroup>-2`, `<hostgroup>-3`, and so on. Numbers increment per host group.

This is deliberate. Host groups are pools of interchangeable VMs, not a hand-managed machine register. No name collisions, no "that name is already taken" errors. Slicer tracks the real hostname internally; your application should track meaning through **tags**.

## Tags for stable identity

Tags are a free-form array of strings attached to each VM. Pass them at creation time:

```bash
curl -X POST http://127.0.0.1:8080/hostgroup/sbox/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "tags": ["user=alice", "job=build-4821", "display=Alice dev environment"],
    "cpus": 2,
    "ram_bytes": 4294967296
  }'
```

Any string is valid. The convention that works well in practice is `key=value`: easy to filter, easy to render in a UI. Use it to carry whatever your application needs to reason about later, for example a sandbox expiry deadline: `expires_at=2026-04-14 08:28:00`.

### Looking up a VM by tag

The list endpoint filters on either exact match or prefix:

```bash
# exact
GET /nodes?tag=user=alice

# prefix (matches any tag starting with "user=")
GET /nodes?tag_prefix=user=
```

Also available on a specific host group:

```bash
GET /hostgroup/sbox/nodes?tag_prefix=user=
```

### How to represent Slicer VMs in your product

Whether your product surfaces VMs to humans (a dashboard, a CLI, a support tool) or to other systems (a scheduler, a billing pipeline, an API), the shape is the same:

1. **Create** with tags carrying the display name, owner, and any internal IDs from your product like tenant, namespace, billing ID, environment, and so on.
2. **List / look up** VMs. On a shared daemon (Slicer per host) scope results with `tag_prefix=owner=` or similar. On a per-tenant daemon the unfiltered list already belongs to one tenant, so a plain `GET /nodes` is enough.
3. **Render** the tag value where an end user sees a VM; keep the auto-assigned hostname as the internal handle your product uses to address it.
4. **Manage** (start, stop, delete) via the real hostname, carried alongside the friendly tag in whatever record your product already stores.

This keeps Slicer's naming model out of your product's domain language while still giving you precise control over each VM.

## Lifecycle

### Ephemeral is the default

VMs launched through the API (`POST /hostgroup/NAME/nodes`) are **ephemeral** by default. They run until one of three things happens, and in every case the disk is removed and there is no automatic restart:

- **DELETE via the API**: the VM stops and the disk is removed.
- **Guest exits on its own** (`sudo reboot`, kernel panic, and similar): the daemon's reaper notices and cleans up the record and the disk.
- **Daemon restart**: ephemeral VM records are not carried across, so the VMs are gone.

This is the right shape for code execution, CI jobs, batch processing, and anything where the VM is disposable.

### Persistent API-launched VMs

For VMs that should survive daemon restarts, such as long-running dev environments, tenant workspaces, and user-facing sandboxes, set `persistent: true` at creation:

```bash
curl -X POST http://127.0.0.1:8080/hostgroup/sbox/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "persistent": true,
    "tags": ["user=alice", "purpose=dev"],
    "cpus": 2,
    "ram_bytes": 4294967296
  }'
```

Or with the CLI:

```bash
slicer vm launch sbox --persistent --tag user=alice
```

Persistent VMs:

- Survive daemon restarts. The daemon re-attaches to them on startup.
- Are **not** deleted when the VM stops. Their disk is retained.
- Can be stopped deliberately without losing state, via `POST /vm/HOSTNAME/shutdown` or `sudo reboot` inside the guest.
- Stay around until you explicitly `DELETE` them through the API or CLI. Delete removes the disk; there is no undelete.

Bring a stopped persistent VM back up with:

```bash
slicer vm relaunch HOSTNAME
```

or the equivalent REST call:

```bash
POST /vm/HOSTNAME/relaunch
```

Relaunch is the intended recovery path whenever a persistent VM has been shut down cleanly, whether by the API, the guest, or a host reboot. Config-declared VMs from `count: N` are a special case: the daemon re-launches them automatically whenever it starts, so no manual `relaunch` is needed for those.

## Where to go next

Most integrations land in one of the shapes below. Pick the row that's closest to what you're building and follow the deployment link.

| Use case | Lifecycle | Deployment | Networking |
| --- | --- | --- | --- |
| Code execution / agent sandbox | Ephemeral | [Slicer per tenant](/platform/instance-per-tenant/) | [Isolated](/reference/networking/) + allowlist |
| CI/CD job runners | Ephemeral | [Slicer per host](/platform/single-instance/) | [Isolated](/reference/networking/) |
| Batch processing | Ephemeral | [Slicer per host](/platform/single-instance/) | Bridge |
| Dev environments | Persistent | [Slicer per tenant](/platform/instance-per-tenant/) | Bridge |
| Tenant workspaces | Persistent | [Slicer per tenant](/platform/instance-per-tenant/) | [Isolated](/reference/networking/) |
| Named resources in your product | Persistent | [Slicer per tenant](/platform/instance-per-tenant/) | Bridge |
| Control plane / shared services | Persistent | [Slicer per host](/platform/single-instance/) | Bridge |

Networking choices are rules of thumb. If your tenants execute untrusted code, default to [isolated mode](/reference/networking/) with an explicit egress allowlist rather than bridge.

## See also

* [Slicer per host](/platform/single-instance/): one daemon, tenants share it, ownership tracked via tags.
* [Slicer per tenant](/platform/instance-per-tenant/): one daemon per tenant, isolated networking, Unix sockets.
* [Go SDK](/platform/go-sdk/)
* [TypeScript SDK](/platform/typescript-sdk/)
* [REST API reference](/reference/api/): exact request/response shapes.
