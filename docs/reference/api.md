# Slicer REST API

This page documents the HTTP API exposed by Slicer for managing micro-VMs, images, disks, and operations.
Some endpoints will fail if a trailing slash is given, i.e. `/nodes` is documented, however `/nodes/` may return an error.

A note on Slicer Services vs Slicer Sandboxes:

* A Slicer Service is a VM launched via YAML - protected from VM deletion - disk persistent is controlled via the `persistent` flag in the YAML config. `storage: image` is persistent by default.
* A Slicer Sandbox is a VM launched via API - and is ephemeral/disposable. It can be deleted, and disk/snapshot is cleaned up on shutdown of the VM or deletion. Shutting down Slicer will terminate and delete all sandboxes.

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

If you intend to expose the Slicer API over the Internet using something like a [self-hosted inlets tunnel](https://inlets.dev/), or [Inlets Cloud](https://cloud.inlets.dev), then make sure you use the "we terminate TLS for you" option.

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

## GET /info

Get server version information.

Response 200:

```json
{ "version": "1.0.0", "git_commit": "abcdef123" }
```

## GET /healthz

Check service liveness.

Response 200:
```json
{ "status": "ok" }
```

## Slicer Agent Health

The VM Agent provides:

* Serial console/Shell sessions
* File copying
* Remote command execution
* Health check/readiness of the agent itself

So before accessing any of these endpoints, it makes sense to first check whether the agent is healthy and accessible, especially if the VM is being booted up via code, then automated immediately afterwards.

For automation, you can use the `HEAD` method to simply check if the agent is started.

To get additional data like System Uptime, Agent Uptime and Agent Version, use a HTTP `GET` method.

`/vm/{hostname}/health`

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
[{"name":"vm","count":1,"ram_bytes":4294967296,"cpus":2,"gpu_count":0,"arch":"x86_64"}]
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

## List nodes in a Host Group

HTTP GET

`/hostgroup/NAME/nodes`

Response 200:

```json
[{"hostname":"vm-1","ip":"192.168.137.2","ram_bytes":2147483648,"cpus":2,"created_at":"2025-09-02T08:32:37.667253315+01:00","arch":"x86_64","status":"Running"}]
```

## Get serial console logs from a VM

HTTP GET

`GET /vm/{hostname}/logs?lines=<n>`

When `n` is empty, a default of 20 lines will be read from the end of the log.

When `n` is `0`, the whole contents will be returned.

Response 200:

```
{"hostname":"vm-1","lines":20,"content":"[  OK  ] Started slicer-ssh-agent.\n[  OK  ] Started slicer-vmmeter.\n"}
```

## Delete a host within a Host Group

HTTP DELETE

`/hostgroup/NAME/nodes/HOSTNAME`

Response 200:

```json
{"message":"Node deleted","disk_removed":"true"}
```

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

## Get node consumption details for a single VM

HTTP GET

`/node/{hostname}/stats`

Response 200:

```json
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
    "memoryUsedPercent": 5.471584459372148
  }
}
```

## Copy files to and from the microVM

HTTP POST/GET

`/vm/{hostname}/cp`

Copy files to/from a VM. This endpoint requires the `slicer-agent` service to be running in the target VM.

Files or directories are always streamed, and not buffered in memory or on disk.

* Files are copied using `binary` mode, where the request or response body is the binary data of the file being copied.
* Directories can be copied using `tar` without compression. When copying folders in either direction, very little metadata is preserved, so you may also want to combine additional options such as `uid` or `gid`, or run your own `chown` via the exec endpoint after the copy.

For the highest level of fidelity, you can create your own tar and copy it as a file. 

### Copy a file or folder to the VM

**Copy a file to the VM**

- Content-Type: `application/octet-stream`
- Body: binary data of file to copy
- Query parameters:
  - `path` (required): destination path in VM
  - `uid` (optional): user ID for file ownership
  - `gid` (optional): group ID for file ownership
  - `mode` (optional): `binary` or `tar` (defaults to `binary`)
  - `permissions` (optional): permissions for copied files (e.g. `0644`)

**Copy a directory to the VM**

- Content-Type: `application/x-tar`
- Body: tar stream of files to copy
- Query parameters:
  - `path` (required): destination path in VM
  - `uid` (optional): user ID for file ownership
  - `gid` (optional): group ID for file ownership
  - `mode` (optional): `tar` (recommended when sending tar content)
  - `permissions` (optional): permissions for copied files (e.g. `0644`)

### Copy a file or folder from the microVM

The same applies as copying a file to the VM, however it works in the reverse. Instead of sending a `Content-Type` header, an `Accept` header is used to indicate whether to use `tar` mode or `binary` mode.

**Copy a file to the client**

- Accept: `application/octet-stream`
- Query parameters:
  - `path` (required): destination path in VM
  - `mode` (optional): `binary` or `tar`
- Response: binary data of file

**Copy a directory to the client**

- Accept: `application/octet-stream`
- Query parameters:
  - `path` (required): destination path in VM
  - `mode` (optional): `tar` (recommended when receiving tar content)
- Response: binary data of directory

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

Body: stdin data (if `stdin=true`)

Response: streaming newline-delimited JSON with stdout/stderr/exit_code

Example response stream:

```json
{"stdout":"Hello from VM\n","stderr":"","exit_code":0}
{"stdout":"","stderr":"some error\n","exit_code":0}
{"stdout":"","stderr":"","exit_code":0}
```

Each JSON object is separated by a newline character.

Note that the `exit_code` variable is an integer so will always be populated with `0`. This is a limitation of JSON serialisation, so to determine the actual exit code, wait until the TCP connection has been disconnected, and keep track of the final exit code value that is received.

The `error` variable may also be populated if there was a problem running, starting, or finding the requested binary or shell to execute. For example:

```json
{"error": "Some error message", "exit_code": 1}
```

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
    "gid": 1000,
    "mod_time": "2025-09-02T08:32:37.667253315+01:00"
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


### Obtain an interactive shell / serial console

HTTP GET

This is endpoint is used by `slicer vm shell` to obtain a shell without needing SSH to be listening in the VM, or to be exposed on any network adapters. It replaces the concept of a serial console from a traditional headless server. No routable access to the VM or its subnet is required, only the REST API of Slicer.

This endpoint is only available in a VM if the `slicer-agent` service is running. The CLI is the only official client compatible with this functionality.

`/vm/{hostname}/shell`

Query parameters:
- `uid` (optional): user ID to run shell as
- `gid` (optional): group ID to run shell as
- `shell` (optional): shell interpreter
- `cwd` (optional): working directory

## Agent health (no body)

HTTP HEAD

`/vm/{hostname}/health`

Response 200: No body

## Shutdown or reboot a VM

HTTP POST

`/vm/{hostname}/shutdown`

Query parameters:
- `action` (optional): `shutdown` or `reboot`

Response 200: text/plain

## Pause a VM

HTTP POST

`/vm/{hostname}/pause`

Response 200: text/plain

## Resume a VM

HTTP POST

`/vm/{hostname}/resume`

Response 200: text/plain

## Forward TCP connections

HTTP GET

`/vm/{hostname}/forward`

This endpoint is used by the inlets client to establish TCP tunnels via the VM agent.

## Prometheus metrics

HTTP GET

`/metrics`
