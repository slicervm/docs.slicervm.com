# Sandboxes

The `sbox` host group is for arm64 Linux microVMs that you launch on demand. Each sandbox gets its own kernel and filesystem.

This is the second half of the Slicer model for Mac:

- **Services** (`slicer`) are your persistent day-to-day VMs.
- **Sandboxes** (`sbox`) are short-lived, disposable environments for tests and agents.

Use sandboxes when you want fast isolation without changing your persistent environment.

```text
+------------------------------------------------------------------+
|                        macOS Host                                |
|                                                                  |
|  +----------------------------+   +---------------------------+  |
|  | sbox-1 (on demand)         |   | sbox-2 (on demand)        |  |
|  | 2 vCPU, 4GB RAM            |   | 2 vCPU, 4GB RAM           |  |
|  |                            |   |                           |  |
|  | opencode + git repo        |   | Docker build + test       |  |
|  | -------------------------- |   | ------------------------- |  |
|  | Copy repo in, run agent,   |   | Copy Dockerfile in,       |  |
|  | copy results out           |   | build, run tests          |  |
|  +----------------------------+   +---------------------------+  |
|                                                                  |
|  +-----------------------------------------------------+         |
|  | slicer-mac daemon (Apple Virtualization)            |         |
|  | slicer.sock (Unix socket API)                       |         |
|  +-----------------------------------------------------+         |
+------------------------------------------------------------------+
```

## Use cases

Sandboxes are designed for short-lived workloads and experimentation:

- **AI coding agents** - copy a repo in, let the agent work, copy results out, then delete if needed.
- **CI and testing** - run builds and repeatable test suites in a clean environment.
- **Untrusted code** - isolate risky runs from your main services.
- **Comparison builds** - run multiple versions of tooling side-by-side without rebuilding your persistent VM.

You can launch sandboxes from the CLI, [Go SDK](/platform/go-sdk/), or [REST API](/reference/api).

## Launch, use, and delete

Launch a sandbox. It keeps its disk until you delete it, even across reboots and restarts of Slicer for Mac:

```bash
slicer vm launch sbox
```

To launch a throwaway sandbox that's discarded as soon as it stops, add `--rm`:

```bash
slicer vm launch sbox --rm
```

Launch a sandbox with memorable tags that you can query later on through the API or `slicer vm list`:

```bash
slicer vm launch sbox \
  --tag owner=alex \
  --tag task=k3s
```

Check what is running:

```bash
slicer vm list
```

Run a command inside the sandbox:

```bash
slicer vm exec sbox-1 -- hostname
```

Open an interactive shell:

```bash
slicer vm shell sbox-1
```

Stop a sandbox and keep its disk:

```bash
slicer vm shutdown sbox-1
```

Delete a sandbox and its disk:

```bash
slicer vm delete sbox-1
```

A sandbox launched with `--rm` is removed as soon as it stops, so there's nothing to delete.

Relaunch a stopped sandbox (after a `slicer vm shutdown` or running `sudo reboot` in the guest):

```bash
slicer relaunch sbox-1
```

## Offline sandboxes

Launch a sandbox without a virtual network device:

```bash
slicer vm add sbox --network none --wait
```

An offline sandbox has no NIC or IP lease and cannot reach the Internet, host network, or Slicer Proxy. Shell, exec, file copy, health checks, and port forwarding remain available over vsock.

Sandboxes use NAT by default. To install dependencies before running offline, [prepare a VM online, commit its disk, and fork it with `--network none`](/platform/forking/#prepare-online-fork-offline). Pass `--network none` for every offline fork, even when its source was offline. Omitting the flag or leaving it empty creates a VM with NAT networking.

Offline networking is not available for the persistent `slicer-1` VM.

## Customise sandbox resources

Bear in mind that `slicer-1` is already set up to use 8GB of RAM by default, so if you only have a 16GB Mac, launching two sandboxes, at 4GB of RAM each, may overcommit your system.

If you only need sandboxes, you can disable `slicer-1` on start-up by setting `count: 0` in the `slicer` hostgroup, or reduce its RAM allocation.

Edit the `sbox` host group in `slicer-mac.yaml` to change the default vCPU, RAM, or disk size for new sandboxes:

```yaml
- name: sbox
  count: 0
  vcpu: 2
  ram_gb: 4
  storage_size: 15G
  rosetta: false
  network:
    mode: nat
    gateway: 192.168.64.1/24
```

Restart the daemon after changing the config.

## Next steps

- [Copy files to/from a VM](/tasks/copy-files) - use `slicer vm cp` to move files in and out of sandboxes
- [Execute commands in a VM](/tasks/execute-commands) - run commands remotely with `slicer vm exec`
- [REST API](/reference/api) - automate sandbox lifecycle via the API
