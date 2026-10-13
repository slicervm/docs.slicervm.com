# Give your agent its own computer

Your coding agent keeps running on your machine, next to your editor and your repository. When it needs somewhere to work, it launches its own microVM with Slicer using the agent skills, and does the work there instead.

Use it to:

* Test new changes you're iterating on end-to-end
* Get a clean-room environment
* Run code with egress auditing or filtering via [Slicer Proxy](/proxy/overview)
* Experiment with new software packages without tainting your machine

The agent gets a real Linux box with root access, systemd, and the internet. Afterwards, the VM is still there for you to look around in.

You can also run the agent itself within a VM, see [Sandbox your agent in a microVM](/getting-started/agent-sandbox).

## Before you start

You'll need Slicer running. New VMs can be launched into any of its host groups. Follow the [walkthrough](/getting-started/walkthrough) up to step 2 if you haven't done so yet. Starting the daemon needs `sudo`, so run that one command yourself rather than asking the agent to.

## Install the agent skills

Slicer's [agent skills](https://github.com/slicervm/agent-skills) teach your agent how to launch VMs, run commands in them, copy files in and out, and clean up afterwards. They work with Claude Code, Codex, Amp, OpenCode, Cursor, and others.

=== "Most agents"

    ```bash
    npx skills add slicervm/agent-skills
    ```

=== "Claude Code plugin"

    ```
    /plugin marketplace add slicervm/agent-skills
    /plugin install use-slicer@slicer
    ```

=== "Ask your agent"

    <div class="agent-prompt" markdown>
    Install slicervm/agent-skills from GitHub
    </div>

## Give it a job

<div class="agent-prompt" markdown>
Spin up a VM using the slicer skill, clone alexellis/arkade into it and build it.
</div>

We gave this exact prompt to an agent running OpenCode with the skill installed. Here's what it ran, in order, lightly trimmed:

```bash
# Check which host groups are available
slicer vm group

# Launch a VM named after the job
slicer vm add vm \
  --name arkade \
  --tag workflow=arkade \
  --wait

# Clone the repository inside the VM
slicer vm exec arkade -- \
  "git clone https://github.com/alexellis/arkade.git"

# Install the Go version that go.mod asks for
slicer vm exec arkade -- \
  "curl -fsSLo go.tar.gz https://go.dev/dl/go1.26.0.linux-amd64.tar.gz && ..."

# Run the build in the background, then wait for it
slicer vm bg exec arkade \
  --cwd /home/ubuntu/arkade \
  --shell=/bin/bash -- \
  "export PATH=/tmp/go/go/bin:$PATH; make build"

slicer vm bg wait arkade $EXEC_ID --timeout 10m

# Check the binary runs
slicer vm exec arkade --cwd ~/arkade -- "./arkade version"
```

It found the host group, launched a named VM, read the Go version from `go.mod`, and installed that toolchain inside the VM. It ran the build as a background job, so a slow build couldn't time out its own shell. Then it checked the binary actually ran:

```
  version: 0.11.132
  build date: 2026-10-13T06:47:29Z
```

Nothing was installed on the machine the agent was running on.

## Check its work

The VM is yours as much as the agent's:

```bash
slicer ls
slicer shell arkade
```

```
ubuntu@vm-2:~$ ~/arkade/arkade version | grep version
  version: 0.11.132
```

VMs are persistent by default, so the agent can come back to the same VM later in the session, and so can you. When you're done, ask the agent to delete the VMs it launched, or run `slicer vm delete arkade`.

## Get the build back

The binaries are still inside the VM. Ask the agent for release builds for every platform, copied back to your machine:

<div class="agent-prompt" markdown>
Run make dist in the arkade VM and copy the bin folder back to ./bin
</div>

It builds them in the VM, then copies the folder out with `slicer cp`, which you can also run yourself:

```bash
slicer cp -r arkade:~/arkade/bin ./bin
ls bin
```

```
arkade  arkade-arm64  arkade-armhf  arkade-darwin  arkade-darwin-arm64  arkade.exe
```

Run the one for your machine, i.e. `./bin/arkade` on Linux, or `./bin/arkade-darwin-arm64` on an Apple Silicon Mac:

```bash
./bin/arkade version
```

```
  commit:  8b8a849ed4f9b54952f2a3b27168648f8fea31f2
  version: 0.11.132
```

Six cross-compiled binaries took just over a minute to build in the VM, and less than a second to copy back.

## More things to ask

<div class="agent-prompt" markdown>
Use Slicer to copy this repo into a new VM, run the test suite there, and tell me what fails.
</div>

<div class="agent-prompt" markdown>
Use Slicer to build this project in a VM and copy the binary back to `./bin`.
</div>

<div class="agent-prompt" markdown>
Use Slicer to launch a VM with Docker installed and forward its socket, so I can run `docker build` locally against it.
</div>

<div class="agent-prompt" markdown>
Use Slicer to set up K3s in a fresh VM, install this Helm chart, and give me the kubeconfig.
</div>

## Tips

* **Name your VMs.** Ask for a VM by name, i.e. "in a VM called tests", and the agent will use `--name`. That makes it easy for both of you to refer to the VM later.
* **Throwaway VMs.** Ask for a throwaway VM if you don't need it afterwards, and the agent will launch it with `--rm`.
* **Long-running work.** The skill uses `slicer vm bg exec` for builds and servers, so they keep running even if the agent's own command times out.
* **Automated workspaces.** If a project has a `slicer.yaml`, the agent can bring the whole workspace up with [`slicer workspace up`](/getting-started/workspaces).
