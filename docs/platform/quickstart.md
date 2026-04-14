# Quickstart: Create a Sandbox via API

This walkthrough covers the full sandbox lifecycle: create a VM, run a command, copy files, and delete it. It assumes Slicer is already [installed](/getting-started/install/) and running.

Each step leads with the `slicer` CLI and then shows the equivalent `curl` call. Both talk to the same REST API.

## Set up a host group

Create a host group with `count: 0` so no VMs are pre-allocated. The API creates them on demand.

```bash
slicer new sandbox \
  --count=0 \
  --graceful-shutdown=false \
  > sandbox.yaml
```

Start Slicer:

```bash
sudo slicer up ./sandbox.yaml
```

For the fastest boot times, add `--storage zfs` or `--storage devmapper` to the `slicer new` command. See [storage backends](/storage/overview/) for setup.

## Get the API token

When authentication is enabled, Slicer writes a bearer token to disk. The CLI reads it automatically; `curl` callers need to supply it themselves:

```bash
export TOKEN=$(sudo cat /var/lib/slicer/auth/token)
```

If you are calling from a remote host, copy the token over SSH:

```bash
export TOKEN=$(ssh user@slicer-host 'sudo cat /var/lib/slicer/auth/token')
```

## Create a sandbox

Block server-side until the in-guest agent is reachable, so the next command can run immediately:

```bash
slicer vm add sandbox --wait --timeout 60s
```

The equivalent `curl` passes `wait=agent` as a query parameter:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  "http://127.0.0.1:8080/hostgroup/sandbox/nodes?wait=agent&timeout=60s" \
  -d '{}'
```

Response:

```json
{
  "hostname": "sandbox-1",
  "hostgroup": "sandbox",
  "ip": "192.168.140.2/24",
  "created_at": "2026-04-11T08:48:41.302713266Z",
  "arch": "amd64",
  "persistent": false
}
```

To override the host group defaults for CPU or RAM, use flags on the CLI:

```bash
slicer vm add sandbox --wait --cpus 4 --ram-gb 8
```

…or add them to the JSON body:

```json
{ "cpus": 4, "ram_bytes": 8589934592 }
```

CPU and RAM requests can match or be lower than the host group's defaults. Requests that exceed the defaults are rejected.

Hostnames are auto-assigned (`sandbox-1`, `sandbox-2`, and so on). Use `tags` to track which VM belongs to which user or job in your system:

```bash
slicer vm add sandbox --wait --tag user=alice --tag job=convert-video-123
```

```json
{ "tags": ["user=alice", "job=convert-video-123"] }
```

Tags come back on every list response. See [VM Lifecycle](/platform/lifecycle/) for how to represent them in your product.

## Run a command

```bash
slicer vm exec sandbox-1 uname -a
```

The equivalent `curl`:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/exec?cmd=uname&args=-a&stdout=true&stderr=true"
```

```json
{"timestamp":"2026-04-11T08:48:47Z","stdout":"Linux sandbox-1 6.1.90 #1 SMP x86_64 GNU/Linux\n"}
```

The response is newline-delimited JSON, streamed as the command runs. The final frame carries the `exit_code`.

For multi-word commands, let the shell parse:

```bash
slicer vm exec sandbox-1 -- "cat /etc/os-release | head -3"
```

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/exec?cmd=bash&args=-c&args=cat+/etc/os-release+|+head+-3&stdout=true"
```

See [execute commands](/tasks/execute-commands/) and the [API reference](/reference/api/#execute-commands) for options including `uid`, `gid`, `cwd`, and `stdin`.

## Copy files in and out

Copy a file into the VM:

```bash
echo "hello from the host" > /tmp/greeting.txt
slicer vm cp /tmp/greeting.txt sandbox-1:/tmp/greeting.txt
```

```bash
echo "hello from the host" | curl -sf \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/cp?path=/tmp/greeting.txt&mode=binary" \
  --data-binary @-
```

Copy the file back out:

```bash
slicer vm cp sandbox-1:/tmp/greeting.txt /tmp/greeting-back.txt
```

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/octet-stream" \
  "http://127.0.0.1:8080/vm/sandbox-1/cp?path=/tmp/greeting.txt&mode=binary"
```

For directories, use `-r` on the CLI or `mode=tar` with `Content-Type: application/x-tar` on the API. See the [API reference](/reference/api/#copy-files-to-and-from-the-microvm) for details on uid, gid, and permissions.

## Create a sandbox with userdata

Run a shell script at first boot, and block until it finishes with `--wait-userdata` (implies `--wait`):

```bash
cat > /tmp/setup.sh <<'EOF'
#!/bin/bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive
apt-get update -qy
apt-get install -qy curl
EOF

slicer vm add sandbox --wait-userdata --timeout 5m --userdata-file /tmp/setup.sh
```

The equivalent API call passes the userdata body and `wait=userdata`:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  "http://127.0.0.1:8080/hostgroup/sandbox/nodes?wait=userdata&timeout=5m" \
  -d "{\"userdata\": $(jq -Rs . < /tmp/setup.sh)}"
```

The daemon holds the response open via a single long-lived HTTP call into the guest agent. When it returns, your userdata has completed and the VM is ready to serve work. No client-side polling required.

## Create a persistent sandbox

By default sandboxes are ephemeral: they are destroyed when Slicer shuts down. To create a VM that survives restarts and retains its disk, pass `--persistent`:

```bash
slicer vm add sandbox --wait --persistent
```

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  "http://127.0.0.1:8080/hostgroup/sandbox/nodes?wait=agent" \
  -d '{"persistent": true}'
```

The `persistent` field is returned in list responses so your application can distinguish between ephemeral and persistent VMs. See [VM Lifecycle](/platform/lifecycle/) for the full semantics.

## Delete the sandbox

```bash
slicer vm delete sandbox-1 sandbox
```

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X DELETE \
  http://127.0.0.1:8080/hostgroup/sandbox/nodes/sandbox-1
```

```json
{"disk_removed":"false","message":"VM deleted successfully"}
```

## Wrapping up

That covers the core loop: create a VM (blocking until it's ready), run commands, move files, delete the VM. Everything here maps to the [REST API reference](/reference/api/) and works the same from any language that can make HTTP calls.

See also:

* [VM Lifecycle](/platform/lifecycle/): ephemeral vs persistent, tags, host group semantics
* [REST API reference](/reference/api/)
* [Go SDK](/platform/go-sdk/)
* [TypeScript SDK](/platform/typescript-sdk/)
* [Authentication](/reference/api/#authentication)
* [Networking](/reference/networking/): CIDR ranges, bridges, and routing
* [Storage backends](/storage/overview/): choosing between image, ZFS, and devmapper
