# Slicer REST API

This page documents the HTTP API exposed by Slicer for managing micro-VMs, images, disks, and operations.

Some endpoints will fail if a trailing slash is given, i.e. `/nodes` is documented, however `/nodes/` may return an error.

## Authentication

No authentication, ideal for local/dev work:

```yaml
config:
  api:
    bind_address: 127.0.0.1
    port: 8080
```

For production:

```yaml
config:
  api:
    bind_address: 127.0.0.1
    port: 8080
    auth:
      enabled: true
```

The token will be saved to: `/var/lib/slicer/auth/token`.

Send an `Authorization: Bearer TOKEN` header for authenticated Slicer daemons.

If you intend to expose the Slicer API over the Internet using something like a [self-hosted inlets tunnel](https://inlets.dev/), or [Inlets Cloud](https://cloud.inlets.dev), then make sure you use the TLS termination option.

i.e.

```bash
DOMAIN=example.com
inletsctl create \
    slicer-api \
    --letsencrypt-domain $DOMAIN \
    --letsencrypt-email webmaster@$DOMAIN
```

Alternatively, if Slicer is on a machine that's public facing, i.e. on Hetzner or another bare-metal cloud, you can use Caddy to terminate TLS. Make sure the API is bound to 127.0.0.1:

Caddyfile:

```
{
    email "webmaster@example.com"
    # Uncomment to try the staging certificate issuer first
    #acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
    # The production issuer:
    acme_ca https://acme-v02.api.letsencrypt.org/directory
 }

slicer-1.example.com {
 reverse_proxy 127.0.0.1:8080
}
```

You can install Caddy via `arkade system install caddy`

## Conventions

- Content-Type: application/json unless noted.
- Timestamps: RFC3339.
- IDs: opaque strings returned by the API.
- Errors: JSON: { "error": "message", "code": "optional_code" } with appropriate HTTP status.

## GET /healthz

Check service liveness.

Response 200:
```json
{ "status": "ok" }
```

## List nodes

HTTP GET

`/nodes`

```json
[{"hostname":"vm-1","ip":"192.168.137.2","created_at":"2025-09-02T08:32:37.667253315+01:00"}]
```

## List Host Groups

HTTP GET

`/hostgroup`

Response 200:

```json
[{"name":"vm","count":1,"ram_gb":4,"vcpu":2}]
```

## Create a new host within a Host Group

HTTP POST

`/hostgroup/NAME/nodes`

Add a host with the defaults:

```json
{
}
```

Or add a host with userdata:

Make sure the string for userdata is JSON encoded.

```json
{
  "userdata": "sudo apt update && sudo apt upgrade"
}

Add a host with a custom GitHub user to override the SSH keys:

```json
{
    "github_user": "your_github_username"
}
```

## Get serial console logs from a VM

HTTP GET

`GET /vm/{hostname}/logs?lines=<n>`

When `n` is empty, a default of 20 lines will be read from the end of the log.

When `n` is `0`, the whole contents will be returned.

Response 200:

```
[  OK  ] Started slicer-ssh-agent.
[  OK  ] Started slicer-vmmeter.
[  OK  ] Reached target Multi-User System.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.
```

## Delete a host within a Host Group

HTTP DELETE

`/hostgroup/NAME/nodes/HOSTNAME`

Response 204: No Content

## Get node consumption details

Providing that the `slicer-vmmeter` service is running in your VM, detailed usage and consumption metrics can be obtained.

HTTP GET

`/nodes/stats`

```json
[
  {
    "hostname": "vm-1",
    "ip": "192.168.137.2",
    "created_at": "2025-09-02T08:32:37.667253315+01:00",
    "snapshot": {
      "hostname": "vm-1",
      "arch": "x86_64",
      "timestamp": "2025-09-02T07:39:32.388239809Z",
      "uptime": "6m53s",
      "totalCpus": 2,
      "totalMemory": 4024136000,
      "memoryUsed": 220184000,
      "memoryAvailable": 3803952000,
      "memoryUsedPercent": 5.471584459372148,
      "loadAvg1": 0,
      "loadAvg5": 0,
      "loadAvg15": 0,
      "diskReadTotal": 62245888,
      "diskWriteTotal": 7815168,
      "networkReadTotal": 44041,
      "networkWriteTotal": 19872,
      "diskIOInflight": 0,
      "openConnections": 0,
      "openFiles": 480,
      "entropy": 256,
      "diskSpaceTotal": 26241896448,
      "diskSpaceUsed": 820826112,
      "diskSpaceFree": 24062115840,
      "diskSpaceUsedPercent": 3.12792222782572
    }
  }
]
```

### Execute a shell

HTTP GET

This is an internal endpoint used by `slicer vm shell` to obtain a shell. It requires the `slicer-agent` service to be running within the guest microVM.

`/vm/{hostname}/shell`

## Copy files

HTTP POST/GET

`/vm/{hostname}/cp`

Copy files to/from a VM. Requires the `slicer-agent` service running in the guest VM and `tar` utility available.

**Copy to VM (POST):**
- Content-Type: `application/x-tar`
- Body: tar stream of files to copy
- Query parameters:
  - `path` (required): destination path in VM
  - `uid` (optional): user ID for file ownership
  - `gid` (optional): group ID for file ownership

**Copy from VM (GET):**
- Query parameters:
  - `path` (required): source path in VM
- Response: tar stream of requested files

## Execute commands

HTTP POST

`/vm/{hostname}/exec`

Execute commands on a VM. Requires the `slicer-agent` service running in the guest VM.

Query parameters:
- `cmd` (required): command to execute
- `args` (optional, multiple): command arguments
- `uid` (optional): user ID to run command as (default: 0)
- `gid` (optional): group ID to run command as (default: 0)
- `cwd` (optional): working directory
- `shell` (optional): shell interpreter (default: /bin/bash)
- `stdin` (optional): enable stdin pipe (true/false)
- `permissions` (optional): file permissions for output

Body: stdin data (if `stdin=true`)

Response: streaming newline-delimited JSON with stdout/stderr/exit_code

Example response stream:

```json
{"stdout":"Hello from VM\n","stderr":"","exit_code":0}
{"stdout":"","stderr":"some error\n","exit_code":0}
{"stdout":"","stderr":"","exit_code":0}
```

Each JSON object is separated by a newline character.

## Manage secrets

HTTP GET

`/secrets`

```json
[
  {
    "name": "my-secret",
    "size": 1024,
    "permissions": "0600",
    "uid": 1000,
    "gid": 1000
  }
]
```

HTTP POST

`/secrets`

Create a secret with base64-encoded data:

```json
{
  "name": "my-secret",
  "data": "base64-encoded-content",
  "permissions": "0600",
  "uid": 1000,
  "gid": 1000
}
```

HTTP PATCH

`/secrets/{name}`

Update an existing secret:

```json
{
  "data": "base64-encoded-content",
  "permissions": "0644",
  "uid": 1001,
  "gid": 1001
}
```

HTTP DELETE

`/secrets/{name}`

Response 200: No content
