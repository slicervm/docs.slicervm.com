# Worktrees

`slicer worktree` lets you push a git repository or worktree into a Slicer microVM, work on it in isolation in the VM or run an agent there, and pull the commits back to your host branch. This is useful when you want to give an agent a fully isolated environment, without losing the ability to commit, review, and push changes from your host under your own identity.

Use `slicer worktree` to:

- [Push a whole repo](#push-a-repo-into-a-vm) — you want to work on or run an agent against your current branch in an isolated VM, then bring the commits back.
- [Push a worktree](#push-a-worktree-into-a-vm) — you want to work on a feature branch in a VM while keeping your main working directory untouched on the host, without stashing or committing unfinished changes.
- [Use an agent sandbox](#use-a-worktree-with-agent-sandbox-commands) — you want to provision a VM with a coding agent installed and then sync a worktree to have the agent work on the code in isolation.

## Push a repo into a VM

Push the current directory into a new VM, open a shell to work in it, then pull commits back:

```bash
cd ~/src/myrepo

# Launch a VM and push the repo in. Take note of the VM name
slicer worktree push --launch .

# Open a shell in the VM and work there
slicer vm shell vm-1

# Pull commits back and fast-forward your host branch
slicer worktree pull vm-1 .

# Push to remote from your host
git push
```

To push into an existing VM instead:

```bash
slicer worktree push vm-1 .
```

!!! note "More than one host group?"
    `--launch` auto-selects the host group only when the daemon has exactly
    one. If several exist — the default on Slicer for Mac, which ships with
    `slicer` and `sbox` — pass `--hostgroup`, e.g.
    `slicer worktree push --launch --hostgroup sbox .`

## Push a worktree into a VM

If you want to work on a feature branch in a VM while keeping your main working directory untouched on the host, create a git worktree first. This lets you continue work on the host without stashing or committing unfinished changes, and gives each branch its own isolated VM.

Create a worktree for a branch, push it into a dedicated VM, and sync commits back:

```bash
cd ~/src/myrepo
# Create a git worktree on a new feature branch
git worktree add ../myrepo-feature -b feature
cd ../myrepo-feature

# Launch a VM and push the worktree in. Take note the VM name
slicer worktree push --launch .

# Open a shell in the VM and work there
slicer vm shell vm-1

# Pull commits back and fast-forward your host branch
slicer worktree pull vm-1 .

# Push to remote from your host
git push
```

To push into an existing VM instead:

```bash
slicer worktree push vm-1 .
```

!!! warning "Don't edit the host worktree while a VM holds it"
    `worktree pull` overwrites host files with the VM's copy. Anything changed on the host since the push will be lost. Treat the host worktree as read-only until you have pulled.

## Use a worktree with agent sandbox commands

Agent commands (`slicer opencode`, `slicer claude`, `slicer codex`, and others) boot a VM and install a coding agent, see [Sandboxing Coding Agents](/examples/coding-agents). For a git repository, pass `--worktree` (alias `--wt`) to provision the VM, push the current worktree in with a working, self-contained `.git`, install the agent into that path, and attach — all in one command:

```bash
cd ~/src/myrepo

# Launch a VM with the agent and the current worktree. Take note of the VM name
slicer opencode --worktree .

# ...let the agent work and commit in the VM...

# Pull commits back and fast-forward your host branch
slicer worktree pull vm-1 .

# Push to remote from your host
git push
```

To reuse a VM you provisioned earlier, push into it explicitly:

```bash
# Provision VM only — coding agent installed, no code copied in
# Take note of the VM name
slicer opencode

# Push the worktree into the VM
slicer worktree push vm-1 .

# Attach and let the agent work
slicer opencode vm-1
```

Passing a plain path to an agent command (e.g. `slicer opencode .`) copies the working directory into the VM but does not have an easy command to re-sync commits back to your host. Use `--worktree`, or `slicer worktree push` for an existing VM, whenever the checkout is a git repository.
