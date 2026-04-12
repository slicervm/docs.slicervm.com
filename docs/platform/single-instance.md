# Slicer Per Host

The simplest deployment is a single Slicer daemon serving all tenants or users. Your application creates VMs on demand and uses tags to track which VM belongs to which user or job.

```
┌───────────────────────────────────────────────────────┐
│  Your Application / SaaS / Internal Tool              │
│                                                       │
│  - receives user request                              │
│  - calls Slicer API to create a sandbox               │
│  - copies files in, runs commands, reads output       │
│  - deletes sandbox when done                          │
│  - returns result to user                             │
└─────────────────────────┬─────────────────────────────┘
                          │ REST API / Go SDK
                          ▼
┌───────────────────────────────────────────────────────┐
│  Slicer Daemon          (bare-metal/nested virt.)     │
│                                                       │
│  ┌───────────────┐ ┌───────────────┐ ┌─────────────┐  │
│  │ sandbox-1     │ │ sandbox-2     │ │ sandbox-n   │  │
│  │ tags:         │ │ tags:         │ │ tags:       │  │
│  │  tenant=a3cf  │ │  tenant=e7d1  │ │  tenant=... │  │
│  │ (microVM)     │ │ (microVM)     │ │ (microVM)   │  │
│  └───────────────┘ └───────────────┘ └─────────────┘  │
│                                                       │
│  KVM + Firecracker        devmapper / ZFS storage     │
└───────────────────────────────────────────────────────┘
```

## When to use this

A single instance is the right starting point when:

* You control who can call the API (your backend is the only client)
* You do not need network isolation between VMs from different users
* You want the simplest possible deployment

All VMs share the same host group, network bridge, and API endpoint. Your application is responsible for only accessing VMs it created, using the hostname returned at creation time.

## Configuration

Generate a config with isolated networking and no pre-allocated VMs:

```bash
slicer new sandbox \
  --net=isolated \
  --count=0 \
  --graceful-shutdown=false \
  --drop 192.168.1.0/24 \
  > sandbox.yaml
```

This produces:

```yaml
config:
  host_groups:
    - name: sandbox
      storage: image
      storage_size: 25G
      count: 0
      vcpu: 2
      ram_gb: 4
      network:
        mode: "isolated"
        drop: ["192.168.1.0/24"]
        allow: ["0.0.0.0/0"]
  image: "ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest"
  hypervisor: firecracker
  graceful_shutdown: false
  api:
    port: 8080
    bind_address: "127.0.0.1"
    auth:
      enabled: true
```

In isolated mode, each VM gets its own network namespace. VMs cannot communicate with each other, with the host, or with the LAN. The `drop` list blocks specific CIDRs (your LAN in this case). See [isolated mode networking](/reference/networking/#isolated-mode-networking) for details.

Start Slicer in its own terminal or tmux window:

```bash
sudo slicer up ./sandbox.yaml
```

For production, run Slicer as a systemd service. See [running in the background](/getting-started/daemon/).

## Using tags to track VMs

Hostnames are auto-assigned (`sandbox-1`, `sandbox-2`, etc.). Pass `tags` when creating a VM to associate it with a user, job, or request in your system:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://127.0.0.1:8080/hostgroup/sandbox/nodes \
  -d '{"tags":["user=alice","job=convert-video-123"]}'
```

Tags are returned when listing VMs, so your application can match VMs back to its own records:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8080/nodes
```

```json
[
  {"hostname":"sandbox-1","tags":["user=alice","job=convert-video-123"],"status":"Running"},
  {"hostname":"sandbox-2","tags":["user=bob","job=run-tests-456"],"status":"Running"}
]
```

If you need API isolation or independent failure domains per tenant, see [instance per tenant](/platform/instance-per-tenant/).

## See also

* [Quickstart](/platform/quickstart/) - curl-based walkthrough of the full VM lifecycle
* [Instance per tenant](/platform/instance-per-tenant/) - stronger isolation with separate daemons
* [Networking](/reference/networking/) - bridge and CIDR configuration
