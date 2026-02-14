# Sandboxes

The `sbox` host group is for ephemeral arm64 Linux VMs that you launch on demand. Each sandbox gets its own kernel and filesystem.

This is the second half of the Slicer model for Mac:

- **Services** (`slicer`) are your persistent day-to-day VMs.
- **Sandboxes** (`sbox`) are short-lived, disposable environments for tests and agents.

Use sandboxes when you want fast isolation without changing your persistent environment.

```text
+------------------------------------------------------------------+
|                        macOS Host                                |
|                                                                  |
|  +----------------------------+   +---------------------------+  |
|  | sbox-1 (ephemeral)         |   | sbox-2 (ephemeral)        |  |
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

## Launch, use, and delete

Launch a sandbox:

```bash
slicer vm launch sbox
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

Stop and remove the sandbox when you are done:

```bash
slicer vm delete sbox-1
```

## Use cases

Sandboxes are designed for short-lived workloads and experimentation:

- **AI coding agents** - copy a repo in, let the agent work, copy results out, then delete if needed.
- **CI and testing** - run builds and repeatable test suites in a clean environment.
- **Untrusted code** - isolate risky runs from your main services.
- **Comparison builds** - run multiple versions of tooling side-by-side without rebuilding your persistent VM.

You can launch sandboxes from the CLI, [Go SDK](/tasks/execute-commands-with-sdk), or [REST API](/reference/api).

## Customise sandbox resources

Edit the `sbox` host group in `slicer-mac.yaml` to change the default vCPU, RAM, or disk size for new sandboxes:

```yaml
- name: sbox
  count: 0
  vcpu: 2
  ram_gb: 4
  storage_size: 15G
  rosetta: true
  network:
    mode: nat
    gateway: 192.168.64.1/24
```

Restart the daemon after changing the config.

## Next steps

- [Copy files to/from a VM](/tasks/copy-files) - use `slicer vm cp` to move files in and out of sandboxes
- [Execute commands in a VM](/tasks/execute-commands) - run commands remotely with `slicer vm exec`
- [Slicer REST API](/reference/api) - automate sandbox lifecycle via the API
