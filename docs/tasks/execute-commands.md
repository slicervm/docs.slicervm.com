# Execute commands in a VM

Just like [copying files in and out of a VM](/tasks/copy-files), executing commands can be done in several ways:

* Initially, via a userdata script or userdata file specified in the host group or via the API/CLI create command after Slicer has been started.
* Through SSH assuming direct network access is available to the VM via the LAN or a VPN.
* Through Slicer's REST API - using your own client, the SDK or the `slicer` CLI.

The exec command allows you to run commands remotely on VMs. The first argument is the instance to run the commands on, `--` optionally separates the slicer CLI options from the command to run on the VM.

```bash
# Run a command as root (default)
slicer vm exec vm-1 whoami
```
Slicer allows you to control the command execution context, including user permissions and working directory.

```bash
# Run command as non-root user (UID 1000)
slicer vm exec vm-1 --uid 1000 whoami
```

Change the working directory and run commands with arguments:

```bash
# Execute command in specific directory
slicer vm exec vm-1 --cwd /var/log -- ls -la
```

Control shell interpretation for complex commands:

```bash
# Use default bash shell for complex commands with pipes
slicer vm exec vm-1 -- "ls -l | sort"

# Use custom shell interpreter
slicer vm exec vm-1 --shell /bin/zsh -- "echo $SHELL"

# Execute directly without shell (faster for simple commands)
slicer vm exec vm-1 --shell "" whoami
```

Combine with local commands using pipes and STDIO:

```bash
# Pipe local file content to VM command
cat /etc/hostname | slicer vm exec vm-1 -- base64 --wrap 9999
```

## Streaming vs buffered exec via the REST API

The `/vm/{hostname}/exec` REST endpoint ships two response shapes:

- **Streaming** (default) — NDJSON frames as output arrives. Preferred
  for long-running commands where you want live stdout/stderr, or when
  you want to measure process-start latency separately from first-byte
  latency using the typed `started` frame.
- **Buffered** — add `buffered=true` to get a single JSON document with
  `stdout`, `stderr`, and `exit_code` once the process exits. Preferred
  for short "run one thing, give me the result" calls; avoids the need
  for client-side NDJSON parsing.

```bash
# Streaming (default)
curl --unix-socket ~/slicer-mac/slicer.sock \
  -X POST "http://localhost/vm/vm-1/exec?cmd=uname&args=-a"

# Buffered
curl --unix-socket ~/slicer-mac/slicer.sock \
  -X POST "http://localhost/vm/vm-1/exec?buffered=true&cmd=uname&args=-a"
# → {"stdout":"Linux vm-1 ...\n","stderr":"","exit_code":0}
```

`buffered=true` does not accept `stdin`; use the streaming endpoint if
you need to pipe stdin data.

## Long-running commands: `bg exec`

`slicer vm exec` blocks the caller until the command exits. For a dev server,
a long build, or any process you want to walk away from, use `slicer bg exec`
(or the longer `slicer vm bg exec`) — the child is detached at launch, survives
client disconnect, and captures its output into an agent-side ring buffer you
can reconnect to at any time.

### Three forms for specifying the command

`bg exec` accepts the command in three forms. All three produce the same
daemon at runtime; pick whichever fits your call site.

**Positional form** — your shell tokenizes. Typical for humans typing at a
prompt:

```bash
slicer bg exec vm-1 --uid 1000 --cwd /home/ubuntu/app -- npm run dev
```

**Explicit form** — deterministic, no shell-quoting concerns. Designed for
scripts, agents, and anywhere you're constructing the argv programmatically:

```bash
slicer bg exec vm-1 --cmd npm --arg run --arg dev
# or short flags:
slicer bg exec vm-1 -c npm -a run -a dev
```

`--cmd`/`-c` is the binary; `--arg`/`-a` is repeatable and order-preserving.
Mutually exclusive with positional form and with `--shell`.

**Shell form** — opt in to shell semantics (`$VAR` expansion, globs, pipes,
`&&`/`||`, chained statements). Pick a shell path with `--shell`/`-s`:

```bash
slicer bg exec vm-1 -s /bin/bash -- "cd /home/ubuntu/app && FOO=bar npm run dev"
```

### Process tree and kill semantics

The shell form wraps the command in `<shell> -lc "<your string>"`. For the
common case of a single command as the script, bash automatically `exec`s the
final command in place of itself — so the process tree is clean and the
daemon is the tracked PID. `slicer bg kill` signals the daemon directly; no
stray shell wrapper.

For scripts where this auto-optimisation doesn't apply (traps, background
jobs, or anything where you want to be sure), write `exec` in your script
yourself:

```bash
slicer bg exec vm-1 -s /bin/bash -- "cd /app && exec ./my-server --flag"
```

The explicit form (`-c`/`-a`) never involves a shell, so the tree is clean
by construction regardless.

Once launched, the `/vm/{hostname}/exec/{exec_id}` family of endpoints —
surfaced as `slicer vm bg <subcommand>` — lets you observe and reap the
process:

```bash
slicer bg list   vm-1                   # table of running + exited-not-reaped
slicer bg info   vm-1 ex_abc123         # JSON status snapshot
slicer bg logs   vm-1 ex_abc123 --follow --from-id 0   # stream
slicer bg kill   vm-1 ex_abc123 --signal TERM --grace 5s
slicer bg wait   vm-1 ex_abc123 --timeout 10m
slicer bg remove vm-1 ex_abc123         # free the ring buffer
```

Same subcommands work under `slicer vm bg <...>` — the top-level `slicer bg`
is a shortcut alias mirroring `slicer ls`, `slicer shell`, `slicer cp`.

Key properties:

* **Detached at launch** — the child is its own session and process group, so
  killing the `slicer` CLI does not kill the child.
* **Ring buffer (1 MiB default)** — stdout + stderr + control frames share the
  buffer. Pass `--ring-bytes 4M` on `vm bg exec` for larger workloads. The
  server enforces an agent-wide 256 MiB cap across all concurrent background
  execs.
* **`from_id` resume** — `vm bg logs --follow --from-id N` picks up at frame
  `N`. If the buffer has evicted history since, the stream emits a
  `type=gap` frame describing how much was lost, then resumes from the
  oldest available id.
* **Explicit reap** — the ring lives until `vm bg remove`. After reap, all
  subsequent info/logs/kill/wait calls return `410 Gone`. Callers that launch
  many execs and never reap them will eventually hit the agent-wide cap.
* **v1 scope: no agent-restart survival** — if the guest agent exits, its
  registry of background execs is lost and the children die with it. Suitable
  for dev servers, CI-style long builds, and AI-agent workflows, not for
  daemonised services across reboots.

See the OpenAPI spec for the underlying REST surface
(`POST /vm/{hostname}/exec?background=true`, plus `GET/DELETE` on
`/vm/{hostname}/exec/{exec_id}` and `/logs`, `/kill`, `/wait-exit`
sub-resources).
