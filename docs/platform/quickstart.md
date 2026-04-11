# Quickstart: Create a Sandbox via API

This walkthrough covers the full sandbox lifecycle using curl: create a VM, wait for it, run a command, copy files, and delete it. It assumes Slicer is already [installed](/getting-started/install/) and running.

## Set up a host group

Create a host group with `count: 0` so no VMs are pre-allocated. The API will create them on demand.

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

When authentication is enabled, Slicer writes a bearer token to disk:

```bash
export TOKEN=$(sudo cat /var/lib/slicer/auth/token)
```

All API calls use this token in the `Authorization` header. If you are calling from a remote host, copy the token over SSH:

```bash
export TOKEN=$(ssh user@slicer-host 'sudo cat /var/lib/slicer/auth/token')
```

## Create a sandbox

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  http://127.0.0.1:8080/hostgroup/sandbox/nodes \
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

To override the host group defaults for CPU or RAM:

```json
{
  "cpus": 4,
  "ram_gb": 8
}
```

## Wait for the agent

The guest agent needs to start before you can run commands or copy files. Poll the health endpoint:

```bash
while true; do
  curl -sf -I -H "Authorization: Bearer $TOKEN" \
    http://127.0.0.1:8080/vm/sandbox-1/health && break
  sleep 1
done
```

A `200` response means the agent is ready. For more detail:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  http://127.0.0.1:8080/vm/sandbox-1/health
```

```json
{
  "hostname": "sandbox-1",
  "agent_uptime": 590528203,
  "agent_version": "0.1.117",
  "userdata_ran": true
}
```

The `userdata_ran` field tells you whether the VM's boot script has finished. This is useful when you pass a `userdata` script at creation time and need to wait for it to complete before proceeding.

## Run a command

Use the exec endpoint to run commands inside the VM:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/exec?cmd=uname&args=-a&stdout=true&stderr=true"
```

```json
{"timestamp":"2026-04-11T08:48:47Z","stdout":"Linux sandbox-1 6.1.90 #1 SMP x86_64 GNU/Linux\n"}
```

The response is newline-delimited JSON, streamed as the command runs. Track the final `exit_code` to determine success.

For multi-word commands, use the `shell` parameter:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/exec?cmd=bash&args=-c&args=cat+/etc/os-release+|+head+-3&stdout=true"
```

See [execute commands](/tasks/execute-commands/) and the [API reference](/reference/api/#execute-commands) for the full set of options including `uid`, `gid`, `cwd`, and `stdin`.

## Copy files in and out

Copy a file into the VM:

```bash
echo "hello from the host" | curl -sf \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/cp?path=/tmp/greeting.txt&mode=binary" \
  --data-binary @-
```

Copy a file back out:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/octet-stream" \
  "http://127.0.0.1:8080/vm/sandbox-1/cp?path=/tmp/greeting.txt&mode=binary"
```

```
hello from the host
```

For directories, use `mode=tar` with `Content-Type: application/x-tar`. See the [API reference](/reference/api/#copy-files-to-and-from-the-microvm) for details on uid, gid, and permissions.

## Create a sandbox with userdata

Pass a shell script as `userdata` to run it on boot:

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  http://127.0.0.1:8080/hostgroup/sandbox/nodes \
  -d "{\"userdata\": $(printf '#!/bin/bash\napt update -qy && apt install -qy curl' | jq -Rs .)}"
```

Poll `/vm/HOSTNAME/health` and check `userdata_ran` to know when the script has finished.

## Delete the sandbox

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X DELETE \
  http://127.0.0.1:8080/hostgroup/sandbox/nodes/sandbox-1
```

```json
{"disk_removed":"false","message":"VM deleted successfully"}
```

## Wrapping up

That covers the core loop: create a VM, wait for the agent, run commands, move files, delete the VM. Everything here maps to the [REST API reference](/reference/api/) and works the same from any language that can make HTTP calls.

See also:

* [REST API reference](/reference/api/) - full endpoint documentation
* [Go SDK](https://github.com/slicervm/sdk) - wraps the REST API for Go programs
* [Authentication](/reference/api/#authentication) - securing the API with bearer tokens
* [Networking](/reference/networking/) - CIDR ranges, bridges, and routing
* [Storage backends](/storage/overview/) - choosing between image, ZFS, and devmapper
