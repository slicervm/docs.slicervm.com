# Coding agents

You can run coding agents like [Claude Code](https://code.claude.com/docs/en/overview), [OpenCode](https://opencode.ai/), and [Amp](https://amp.dev/) inside your Slicer for Mac VM. VirtioFS sharing means the agent edits code on a shared folder that's visible on both your Mac and the VM, so you get the isolation of a VM without copying files around.

Running an agent inside a VM means you don't have to pass `--dangerously-skip-permissions` on your Mac. The agent gets its own kernel and filesystem, and can't touch your SSH keys, cloud credentials, or browser sessions.

## Automated agent and sandbox launches

Slicer has a number of experimental commands that do the following:

* Boot a new VM
* Copy the current directory to the VM in the same relative path
* Installs the specified AI agent, and copies in its config/auth files
* Launches the agent in a shell (`slicer vm shell`)

Available agent commands:

* `slicer amp`
* `slicer copilot`
* `slicer codex`
* `slicer claude`
* `slicer opencode`

Finally, there's another option for custom AI agents `slicer workspace`

To launch i.e. a new slicer VM for Claude:

```
mkdir -p ~/dev/
cd ~/dev/
git clone --depth=1 https://github.com/alexellis/arkade

cd arkade

slicer claude .
```

When launching a VM for a coding agent, you can specify:

* `--tmux=none` - launch the agent directly in the shell
* `--tmux=local` - launch a shell using tmux on your mac (requires `brew install tmux`)
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

Both binaries are installed into `/usr/local/bin` inside the VM. They persist across reboots since `slicer-1` is a persistent VM.

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
cd /home/ubuntu/host/my-project
claude --dangerously-skip-permissions
```

Inside the VM, `--dangerously-skip-permissions` is acceptable because the VM is the sandbox.

!!! note "Claude Max plan"
    The Claude Max subscription uses OAuth-based authentication. Run `claude` inside the VM and follow the interactive login flow. The auth token is stored inside the VM and does not affect your Mac.

### OpenCode

Authenticate with your provider of choice:

```bash
opencode auth login --provider github --token <your-github-token>
```

The auth config is stored at `~/.local/share/opencode/auth.json` inside the VM.

## Work on shared folders

With [folder sharing](/mac/folder-sharing), your Mac project paths can be visible at `/home/ubuntu/host/`.

```bash
cd /home/ubuntu/host/code/my-project

# Run Claude Code
claude --dangerously-skip-permissions

# Or run OpenCode
opencode
```

Changes the agent makes are immediately visible on your Mac because VirtioFS is a shared mount, not a copy.

### Share a sub-path for tighter isolation

If you don't want the agent to see your entire home directory, set `share_home` to a sub-path in `slicer-mac.yaml`:

```yaml
share_home: "~/code/"
```

Restart the daemon after changing this setting. The agent can only see projects under `~/code/`, not your SSH keys, cloud credentials, or dotfiles.

## Run agents in sandboxes

For one-off tasks, launch a sandbox instead of using your persistent VM:

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

- [Sandboxes](/mac/launch-sandboxes) - more on launching and managing ephemeral VMs
- [Headless OpenCode on Slicer for Linux](/examples/opencode-agent) - one-shot agent execution via userdata and the exec API
- [Headless Cursor CLI on Slicer for Linux](/examples/cursor-cli-agent) - running Cursor's CLI agent headlessly
- [Trying Claude Code with Ollama](https://slicervm.com/blog/trying-claude-code-with-ollama) - using Claude Code with a local LLM backend
