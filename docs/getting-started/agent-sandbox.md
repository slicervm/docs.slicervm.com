# Sandbox your agent in a microVM

Run your coding agent inside its own microVM, rather than on your machine. The agent can install packages, run any command, and use its most permissive mode, without being able to touch your SSH keys, cloud credentials, browser sessions, or anything else on your machine.

When it's done, you pull its commits back and push them yourself, under your own identity.

You can also keep the agent on your machine and have it launch VMs for its work, see [Give your agent its own computer](/getting-started/agent-computer).

## Before you start

You'll need Slicer running. New VMs can be launched into any of its host groups. Follow the [walkthrough](/getting-started/walkthrough) up to step 2 if you haven't done so yet. You'll also need the agent installed and logged in on your machine, because Slicer copies its credentials into the VM.

## Pick your agent

There's one command per agent, and they all take the same flags:

| Command | Agent |
|---------|-------|
| `slicer claude` | [Claude Code](https://code.claude.com/docs/en/overview) |
| `slicer codex` | [Codex](https://github.com/openai/codex) |
| `slicer opencode` | [OpenCode](https://opencode.ai/) |
| `slicer amp` | [Amp](https://ampcode.com/) |
| `slicer copilot` | [GitHub Copilot CLI](https://github.com/github/copilot-cli) |
| `slicer pi` | Pi |
| `slicer workspace` | No agent, just a shell, for any other tool |

There are two ways to start: with just the agent, or with the agent and your repository.

## Just the agent

Run the agent's command on its own:

```bash
slicer opencode
```

```
- [ok] Prepare cached opencode binary (v1.19.0, cache hit, 177.14 MiB) (210ms)
- [ok] Discover host groups (0s)
- [ok] Create VM in vm (870ms)
- [ok] Wait for VM to become ready: vm-4 (1s)
- [ok] Sync git config to vm-4 (10ms)
- [ok] Copy auth to vm-4 (0s)
- [ok] Copy config to vm-4 (0s)
- [ok] Copy plugins to vm-4 (0s)
- [ok] Copy cached opencode to vm-4 (1.01s)
- [ok] Prepare bootstrap script in vm-4 (10ms)
Provisioned VM: vm-4 (no workspace copied)
Push a worktree: slicer wt push vm-4 .
Attach:          slicer opencode vm-4
To delete:       slicer vm delete vm-4
```

You get a fresh VM with the agent installed and logged in, and nothing else copied from your machine. Attach to it whenever you're ready:

```bash
slicer opencode vm-4
```

This is the quickest way to give an agent a machine to itself. Use it for research, trying out tools, scripting, or anything that isn't tied to a repository on your machine. Add `--name` to give the VM a name you'll remember.

## With your repository

From a git repository, add `--worktree .`:

```bash
cd ~/src/arkade
slicer opencode --worktree . --name arkade-sbx
```

```
- [ok] Prepare cached opencode binary (v1.19.0, cache miss, 177.14 MiB) (1.57s)
- [ok] Discover host groups (0s)
- [ok] Create VM in vm (1.18s)
- [ok] Wait for VM to become ready: vm-3 (1s)
- [ok] Sync git config to vm-3 (20ms)
- [ok] Copy auth to vm-3 (10ms)
- [ok] Copy config to vm-3 (0s)
- [ok] Copy plugins to vm-3 (0s)
- [ok] Push worktree to vm-3 (490ms)
- [ok] Copy cached opencode to vm-3 (1.01s)
- [ok] Prepare bootstrap script in vm-3 (10ms)
```

Around six seconds later, you're in the agent's session, running inside the VM in the same path as on your machine:

<div class="terminal">
<div class="terminal-bar"><span></span><span></span><span></span><em>slicer opencode --worktree . --name arkade-sbx</em></div>
<pre>
                                  <span class="prompt">opencode</span>

  ┃  Ask anything… "What is the tech stack of this project?"
  ┃
  ┃  Build · deepseek-v4-flash-0731 toilgate
  ╹▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                                              tab agents  ctrl+p commands
  <span class="muted">~/src/arkade:master</span></pre>
</div>

`--worktree` copies the repository in with a working, self-contained `.git`, so the agent can commit as it goes. Your git name and email are copied in too, but never your git credentials.

### Give it a task

We asked the agent to make a small change and commit it:

<div class="agent-prompt" markdown>
Add a line under the title in README.md that says: Built in a Slicer sandbox. Then commit it with the message: Note the Slicer sandbox build
</div>

The commit is made inside the VM. Your own checkout hasn't changed yet.

### Bring the commits back

From your checkout, pull the sandbox's commits back. Your branch is fast-forwarded to match:

```bash
slicer wt pull arkade-sbx .
git log --oneline -2
```

```
ce3da61 Note the Slicer sandbox build
8b8a849 Install Codex with its companion package
```

Review the changes, then push them from your machine as you normally would:

```bash
git show --stat HEAD
git push
```

## Come back later

Sandboxes are persistent by default. Reattach to the agent's session by name, or open a shell alongside it:

```bash
slicer opencode arkade-sbx
slicer shell arkade-sbx
```

Delete the sandbox once you've pulled what you need:

```bash
slicer vm delete arkade-sbx
```

Pass `--rm` when you launch a sandbox to make it throwaway, so it's deleted when it stops.

## Without a git repository

To copy a plain directory in instead, pass its path without `--worktree`, i.e. `slicer claude ./my-project`. Files matching a `.slicerignore` at its root are left out. Copy results back out with `slicer cp -r`.

## Next steps

* [Sandboxing coding agents](/examples/coding-agents) covers authentication, installing other agents, and running agents headless.
* [Worktrees](/tasks/worktrees) covers `slicer wt push` and `pull` in more depth.
* [Slicer Proxy](/proxy/overview) limits which sites the agent can reach, and [injects API keys](/proxy/secret-injection) on the way out so they never enter the VM.
