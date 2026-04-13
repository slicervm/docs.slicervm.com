# TypeScript SDK

The [Slicer TypeScript SDK](https://github.com/slicervm/sdk-ts) wraps the REST API into a typed Node.js client, published as [`@slicervm/sdk`](https://www.npmjs.com/package/@slicervm/sdk). It covers the full VM lifecycle: creating and deleting VMs, executing commands, copying files, managing secrets, and polling for readiness.

```bash
npm install @slicervm/sdk
```

The SDK ships dual ESM + CommonJS builds with full TypeScript types. Node 18 or later is required.

## Quickstart

Create a VM, run a command, delete the VM:

```ts
import { SlicerClient, GiB } from '@slicervm/sdk';

const client = SlicerClient.fromEnv(); // reads SLICER_URL + SLICER_TOKEN

const vm = await client.vms.create(
  'sbox',
  { cpus: 1, ramBytes: GiB(1) },
  { wait: 'agent', waitTimeoutSec: 60 },
);
console.log(`created ${vm.hostname} (${vm.ip})`);

try {
  const result = await vm.execBuffered({ command: 'uname', args: ['-a'] });
  console.log(`exit=${result.exitCode}`);
  console.log(result.stdout.trim());
} finally {
  await vm.delete();
}
```

Run it:

```bash
SLICER_URL=~/slicer-mac/slicer.sock npx tsx run-command.ts
```

Expected output:

```
created sbox-1 (192.168.64.3)
exit=0
Linux sbox-1 6.12.70 #1 SMP aarch64 GNU/Linux
```

The complete runnable source — with logging, env handling, and a remote-daemon variant — is in [`examples/run-command/`](https://github.com/slicervm/sdk-ts/tree/master/examples/run-command).

## Shape

The SDK has two layers:

**Control plane — namespaced on `SlicerClient`.** These operate on the Slicer server itself, not any particular VM:

* `client.hostGroups` — list host groups, list VMs within a group.
* `client.vms` — create, attach, list, and collect stats for VMs across host groups.
* `client.secrets` — create, list, update, delete secrets.

**Per-VM operations — on the `VM` handle returned from `client.vms.create()` or `client.vms.get()`.** Everything you do to a single VM is a method on its handle:

* `vm.exec()` / `vm.execBuffered()` — run commands.
* `vm.fs.*` — read/write files, list directories, tar transfer.
* `vm.pause()` / `resume()` / `suspend()` / `restore()` / `shutdown()` / `relaunch()` — lifecycle.
* `vm.health()` / `waitForAgent()` / `logs()` — observability.
* `vm.delete()` — teardown.

This two-layer split keeps control-plane operations (which act on the server) cleanly separate from per-VM operations (which act on a specific VM), making the API easier to navigate as the surface grows.

## Examples

Runnable examples live in [`examples/` in the SDK repo](https://github.com/slicervm/sdk-ts/tree/master/examples). Each is a self-contained TypeScript project — `npm install && npm start`.

* [`run-command`](https://github.com/slicervm/sdk-ts/tree/master/examples/run-command) — canonical hello-world: create a VM, run `uname -a`, delete the VM. The same code is shown inline in the Quickstart above.
* [`nginx`](https://github.com/slicervm/sdk-ts/tree/master/examples/nginx) — install nginx via userdata, port-forward host `127.0.0.1:8080` → VM `:80`, fetch the welcome page from the host through the tunnel. Full create-to-fetch-to-teardown in ~6 seconds.
* [`k3s`](https://github.com/slicervm/sdk-ts/tree/master/examples/k3s) — provision a single-node K3s cluster via userdata + `k3sup`, port-forward `:6443` to a local port, rewrite the kubeconfig, and run host-side `kubectl get nodes` against it. Mirrors the [Go k3s example](https://github.com/slicervm/sdk/tree/main/examples/k3s-userdata) but uses port-forwarding so guest IP routability is irrelevant.
* [`ffmpeg`](https://github.com/slicervm/sdk-ts/tree/master/examples/ffmpeg) — full walkthrough on the [Video conversion (TypeScript)](/platform/typescript-video-conversion/) page. Streams a video into the VM via `vm.fs.writeFile`, transcodes with ffmpeg, streams the result back through `execBuffered({ stdio: 'base64' })` — no intermediate file on the guest disk.

## Connecting to the API

Pass a base URL and bearer token:

```ts
import { SlicerClient } from '@slicervm/sdk';

const client = new SlicerClient({
  baseURL: 'http://127.0.0.1:8080',
  token: process.env.SLICER_TOKEN,
});
```

The token is saved at `/var/lib/slicer/auth/token` when API authentication is enabled. See [authentication](/reference/api/#authentication) for setup.

The SDK also supports UNIX sockets. Pass an absolute socket path or `unix://` URL instead of an HTTP URL:

```ts
const client = new SlicerClient({ baseURL: '/var/run/slicer/api.sock' });
```

Or read `SLICER_URL` and `SLICER_TOKEN` from the environment:

```ts
const client = SlicerClient.fromEnv();
```

## Create a VM

Create a VM in a host group. Pass an empty request to use the host group defaults, or override CPU, RAM, tags, or userdata:

```ts
import { SlicerClient, GiB } from '@slicervm/sdk';

const vm = await client.vms.create('sandbox', {
  cpus: 4,
  ramBytes: GiB(4),
  userdata: '#!/bin/bash\napt update -qy && apt install -qy curl',
});

console.log(`hostname=${vm.hostname} ip=${vm.ip}`);
```

`client.vms.create()` returns a `VM` handle. All per-VM operations hang off this handle rather than taking a hostname argument every time.

## Wait for the agent

The guest agent starts after the VM boots. You can either block server-side by passing `wait: 'agent'`, or poll from the client with `vm.waitForAgent()`:

```ts
// Server-side wait — the daemon holds the response open until the agent is
// reachable, then returns a ready VM handle.
const vm = await client.vms.create(
  'sandbox',
  { cpus: 2, ramBytes: GiB(2) },
  { wait: 'agent', waitTimeoutSec: 60 },
);

// Or poll client-side.
const vm2 = await client.vms.create('sandbox', {});
await vm2.waitForAgent({ timeoutMs: 60_000, intervalMs: 250 });
```

If you passed userdata and need to wait for the script to finish, use `wait: 'userdata'` (server-side) or `vm.waitForUserdata()` (client-side).

## Run commands

The SDK provides two ways to execute commands. `execBuffered()` returns a single result once the command exits:

```ts
const r = await vm.execBuffered({ command: 'uname', args: ['-a'] });
console.log(r.stdout, 'exit=', r.exitCode);
```

For streaming output, `exec()` returns an async iterator of NDJSON frames (`started`, `stdout`, `stderr`, `exit`):

```ts
for await (const f of vm.exec({
  command: '/bin/sh',
  args: ['-c', 'apt update && apt install -y nginx'],
})) {
  if (f.type === 'stdout') process.stdout.write(f.stdout ?? f.data ?? '');
  if (f.type === 'exit') console.log('exit', f.exitCode);
}
```

For binary output (video, archives, raw protocol streams), pass `stdio: 'base64'`. The SDK auto-decodes the wire format and returns `Buffer` values:

```ts
const r = await vm.execBuffered({
  command: '/bin/sh',
  args: ['-c', 'tar -cf - /var/log'],
  stdio: 'base64',
});
// r.stdout is a Buffer when stdio === 'base64'
require('node:fs').writeFileSync('./logs.tar', r.stdout);
```

Streaming `exec()` with `stdio: 'base64'` populates `frame.stdoutBytes` / `frame.stderrBytes` / `frame.dataBytes` as decoded Buffers alongside the base64 string fields.

`exec()` supports `stdin` for commands that read input; `execBuffered()` does not — use the streaming variant for stdin.

## File operations

Per-VM filesystem operations live on `vm.fs`:

```ts
await vm.fs.mkdir({ path: '/data', recursive: true });
await vm.fs.writeFile('/data/input.txt', 'hello');

const entries = await vm.fs.readDir('/data');
const bytes = await vm.fs.readFile('/data/input.txt'); // Buffer

const info = await vm.fs.stat('/data/input.txt');
const present = await vm.fs.exists('/data/input.txt');

await vm.fs.remove('/data/input.txt');
await vm.fs.remove('/data', true); // recursive
```

For directory uploads/downloads, use tar mode:

```ts
const tar = await vm.fs.tarFrom('/etc');
await vm.fs.tarTo('/tmp/restored', tar);
```

See [copy files](/reference/api/#copy-files-to-and-from-the-microvm) in the API reference for the underlying HTTP endpoints.

## Delete a VM

```ts
await vm.delete();
```

Use `try/finally` to clean up VMs on program exit:

```ts
const vm = await client.vms.create('sandbox', { cpus: 1, ramBytes: GiB(1) }, { wait: 'agent' });
try {
  // ... work ...
} finally {
  await vm.delete();
}
```

## Pause, resume, and snapshots

Pause a VM to free CPU while keeping its state in memory, then resume when needed:

```ts
await vm.pause();
// later...
await vm.resume();
```

Suspend writes a snapshot to disk (Slicer for Mac; Linux support in progress). `relaunch()` restarts an API-created VM from its existing disk (requires `persistent: true` at creation time):

```ts
await vm.suspend();
await vm.restore();

await vm.shutdown();
await vm.relaunch();
```

## Secrets

```ts
await client.secrets.create({ name: 'github-token', data: process.env.GH_TOKEN });
const list = await client.secrets.list();
await client.secrets.patch('github-token', { data: newValue });
await client.secrets.delete('github-token');
```

The SDK base64-encodes secret data transparently; pass plaintext.

## Method reference

### Top-level

| Method | Description |
|--------|-------------|
| `new SlicerClient({ baseURL, token?, userAgent? })` | Create a client. `baseURL` accepts `http(s)://…` or a Unix socket path |
| `SlicerClient.fromEnv()` | Build from `SLICER_URL` + `SLICER_TOKEN` |
| `client.getInfo()` | Platform, arch, version |

### Host groups

| Method | Description |
|--------|-------------|
| `client.hostGroups.list()` | List host groups |
| `client.hostGroups.find(name)` | Lookup a host group by name |
| `client.hostGroups.listVMs(name, { tag?, tagPrefix? })` | List VMs in a host group |

### VMs

| Method | Description |
|--------|-------------|
| `client.vms.create(group, req, opts?)` | Create a VM, returns a `VM` handle |
| `client.vms.get(hostname)` | Lookup a VM by hostname |
| `client.vms.attach(group, hostname)` | Build a `VM` handle without an API call |
| `client.vms.list({ tag?, tagPrefix? })` | List all VMs |
| `client.vms.stats()` | Per-VM CPU / memory / disk stats |

### VM handle — `vm.*`

| Method | Description |
|--------|-------------|
| `vm.exec(req)` | Streaming exec, yields NDJSON frames |
| `vm.execBuffered(req)` | Buffered exec, returns a single result |
| `vm.health()` | Agent readiness |
| `vm.waitForAgent(opts?)` | Poll until the agent responds |
| `vm.waitForUserdata(opts?)` | Poll until userdata completes |
| `vm.logs()` | Serial console logs |
| `vm.pause() / resume()` | Pause or resume the VM in memory |
| `vm.suspend() / restore()` | Snapshot to/from disk (Mac) |
| `vm.shutdown(req?) / relaunch()` | Shut down or relaunch a persistent VM |
| `vm.delete()` | Delete the VM |
| `vm.fs.*` | Filesystem operations (below) |

### VM filesystem — `vm.fs.*`

| Method | Description |
|--------|-------------|
| `readFile(path)` / `writeFile(path, bytes)` | Binary file transfer |
| `readDir(path)` / `stat(path)` / `exists(path)` | Directory listing and metadata |
| `mkdir({ path, recursive?, mode? })` / `remove(path, recursive?)` | Create or delete |
| `tarFrom(path)` / `tarTo(path, tar)` | Directory transfer via tar |

### Secrets

| Method | Description |
|--------|-------------|
| `client.secrets.list()` | List secrets (metadata only) |
| `client.secrets.create({ name, data, permissions?, uid?, gid? })` | Create a secret |
| `client.secrets.patch(name, { data, ... })` | Update a secret |
| `client.secrets.delete(name)` | Delete a secret |

## See also

* [SDK source on GitHub](https://github.com/slicervm/sdk-ts) — issues, releases, changelog
* [`@slicervm/sdk` on npm](https://www.npmjs.com/package/@slicervm/sdk)
* [Go SDK](/platform/go-sdk/) — same API, idiomatic Go
* [REST API reference](/reference/api/) — the underlying HTTP endpoints
* [Quickstart](/platform/quickstart/) — curl-based walkthrough of the same lifecycle
