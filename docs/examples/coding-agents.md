# Sandboxing Coding Agents

You can run coding agents like [Claude Code](https://code.claude.com/docs/en/overview), [OpenCode](https://opencode.ai/), and [Amp](https://amp.dev/) inside Slicer VMs.
This page applies to both Slicer for Linux and Slicer for Mac.

Running an agent inside a VM means you don't need to grant broad permissions on your host CLI. The agent gets its own kernel and filesystem, and can't touch your SSH keys, cloud credentials, or browser sessions.

## `slicer workspace`: generic workspace runner

`slicer workspace` is a generic command that bootstraps a fresh VM for your current project workspace. It copies your working directory into the VM and drops you into a shell so you can run a tool of your choice.

Use it when you want one-VM-per-task behavior without being tied to a specific agent binary. For example, use it for:

* trying out a new tool against a codebase
* running a custom script-based agent
* reproducing task-specific work in a disposable environment

Because it is generic, it’s a good fit when the built-in `slicer claude`, `slicer opencode`, etc. shortcuts don’t match your exact workflow.

## Automated agent and sandbox launches

Slicer has a number of experimental commands that do the following:

* Boot a new VM
* Copy the current directory to the VM in the same relative path
* Install the specified AI agent and copy in its config/auth files
* Launch the agent in a shell (`slicer vm shell`)

Available agent commands:

* `slicer workspace`
* `slicer amp`
* `slicer copilot`
* `slicer codex`
* `slicer claude`
* `slicer opencode`

If your agent (for example, Pi or Hermes) is not listed above, request support in the Discord, or use `slicer workspace` and install and configure your agent via a bash script or userdata.

To launch a new Slicer VM for Claude:

```bash
mkdir -p ~/dev/
cd ~/dev/
git clone --depth=1 https://github.com/alexellis/arkade

cd arkade

slicer claude .
```

When launching a VM for a coding agent, you can specify:

* `--tmux=none` - launch the agent directly in the shell
* `--tmux=local` - launch a shell using tmux on the host (requires `brew install tmux` on macOS)
* `--tmux=remote` - launch a shell using tmux on the VM

Tmux is a time-tested terminal tool and ideal for running processes in the background, and reconnecting to them later.

## Install agents manually

Shell into your VM and install agents with [arkade](https://github.com/alexellis/arkade):

```bash
slicer vm shell slicer-1

# Install Claude Code
arkade get claude --path /usr/local/bin

# Install OpenCode
arkade get opencode --path /usr/local/bin
```

Both binaries are installed into `/usr/local/bin` inside the VM. They persist across reboots if your base VM is persistent.

## Authenticate

### Claude Code with an API key

Set the `ANTHROPIC_API_KEY` environment variable inside the VM:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

Add it to `~/.bashrc` to persist across sessions:

```bash
echo 'export ANTHROPIC_API_KEY=sk-ant-...' >> ~/.bashrc
```

Then run Claude Code:

```bash
cd /home/ubuntu/host/code/my-project
claude --dangerously-skip-permissions
```

Inside the VM, `--dangerously-skip-permissions` is acceptable because the VM is the sandbox.

!!! note "Claude Max plan"
    The Claude Max subscription uses OAuth-based authentication. Run `claude` inside the VM and follow the interactive login flow. The auth token is stored inside the VM and does not affect your host.

### OpenCode

Authenticate with your provider of choice:

```bash
opencode auth login --provider github --token <your-github-token>
```

The auth config is stored at `~/.local/share/opencode/auth.json` inside the VM.

## Linux-specific workspace details

On Slicer for Linux, use the workspace mounting pattern defined for your host/group and the normal `slicer vm cp` flow for file movement.

Use:
- your host filesystem access model (from your host/group config)
- `slicer vm cp` for explicit copies into and out of ephemeral VMs

## Run headless agents non-interactively

For one-off and automated tasks, launch a sandbox instead of using your persistent VM:

```bash
# Launch a sandbox
slicer vm launch sbox

# Copy a repo in
slicer vm cp ./my-project sbox-1:/home/ubuntu/my-project

# Run the agent
slicer vm exec --uid 1000 --cwd /home/ubuntu/my-project sbox-1 -- \
  claude --dangerously-skip-permissions -p "Write tests for main.go"

# Copy results out
slicer vm cp sbox-1:/home/ubuntu/my-project ./my-project-result

# Delete when done
slicer vm delete sbox-1
```

The sandbox has its own kernel and filesystem. If the agent does something unexpected, delete it and start fresh.

## Next steps

- [Sandboxes](/mac/sandboxes) - more on launching and managing ephemeral VMs
- [Headless OpenCode on Slicer for Linux](/examples/opencode-agent) - one-shot agent execution via userdata and the exec API
- [Headless Cursor CLI on Slicer for Linux](/examples/cursor-cli-agent) - running Cursor's CLI agent headlessly
- [Trying Claude Code with Ollama](https://slicervm.com/blog/trying-claude-code-with-ollama) - using Claude Code with a local LLM backend
