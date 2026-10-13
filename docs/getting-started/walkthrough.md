# Launch Slicer for the first time on Linux

In this guide you'll start Slicer and launch a microVM on demand by name. Then you'll work inside it, and see how to hand the same workflow over to your coding agent. It takes around five minutes on an x86_64 or Arm64 host.

Each VM is a real Linux machine with its own Kernel and systemd. Docker, Kubernetes and anything else you'd run on a server will just work. A new VM boots in a couple of seconds.

## Before you start

[Install Slicer](/getting-started/install) and save the license key from your email to `~/.slicer/LICENSE`. Then [join the `slicer` group](/getting-started/install#run-slicer-commands-without-sudo) so you don't need `sudo` for every command.

## Give your agent its own computer

Slicer's [agent skills](https://github.com/slicervm/agent-skills) teach your coding agent how to launch its own VMs and work inside them, instead of on your machine. They work with Claude Code, Codex, Amp, OpenCode, Cursor and others.

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

Once Slicer is running, ask your agent for a VM:

<div class="agent-prompt" markdown>
Spin up a VM using the slicer skill, clone alexellis/arkade into it and build it.
</div>

The agent launches a VM, installs what the build needs, and works inside it. Your machine stays untouched, and the VM is still there for you to `slicer shell` into afterwards. See [Give your agent its own computer](/getting-started/agent-computer) for what it runs, or [Sandbox your agent in a microVM](/getting-started/agent-sandbox) to run the agent itself inside a VM.

If you haven't started Slicer yet, follow the steps below first. Starting the daemon needs `sudo`, so it's best to run that one command yourself.

!!! tip "The short version"
    In one terminal, create a config and start Slicer:

    ```bash
    mkdir -p ~/slicer && cd ~/slicer
    slicer new vm --count 0 > slicer.yaml
    sudo slicer up
    ```

    In a second terminal, launch a VM and get a shell:

    ```bash
    slicer vm add vm --name dev
    slicer ready dev
    slicer shell dev
    ```

    Read on for what each step does, and what to try next.

## How Slicer is organised

Slicer runs as a daemon on your host. Its default config file is called `slicer.yaml`, and you can pass a different path to `slicer up` if you need to.

The config defines one or more **host groups**. A host group is a template for VMs with the vCPU, RAM, disk, network range and OS image to use.

Most of the time you'll launch VMs on demand by name with `slicer vm add`. This is how coding agents, sandboxes and dev environments use Slicer. It's what `count: 0` is for - Slicer starts with no VMs and you add them as you need them. This guide uses `count: 0`.

Setting `count` higher launches that many VMs every time Slicer starts. That's best for appliances and clusters that should always be running, such as a [PiHole](/examples/pihole-adblock) or the nodes of a [Kubernetes cluster](/examples/ha-k3s).

## 1. Create a config for the daemon

Create a working directory for Slicer. Your config and each VM's disk image will be stored here.

The first argument to `slicer new` is the name of the host group, in this case `vm`:

```bash
mkdir -p ~/slicer
cd ~/slicer

slicer new vm --count 0 > slicer.yaml
```

The host group's name is used in three places:

* As the prefix for each VM's hostname i.e. `vm-1`, `vm-2`
* When launching a VM into that group i.e. `slicer vm add vm`
* To derive each VM's MAC address, so every host group on a machine needs a unique name

We've used `vm` to keep this guide generic. Feel free to name it after what it's for instead, such as `sandbox`, `agents` or `k3s`.

Take a look at what was generated:

```bash
cat slicer.yaml
```

```yaml
config:
  host_groups:
    - name: vm
      storage: image
      storage_size: 25G
      persistent: true
      count: 0
      vcpu: 2
      ram_gb: 4
      network:
        bridge: brvm0
        tap_prefix: vm
        gateway: 192.168.137.1/24
  image: "ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest"
  hypervisor: firecracker
  api:
    port: 8080
    bind_address: "127.0.0.1"
    auth:
      enabled: true
```

| Field | Value | Description |
|-------|-------|-------------|
| `count` | `0` | No VMs at start-up. Launch them on demand. |
| `vcpu` / `ram_gb` | `2` / `4` | The default size of each VM, and the most any one VM in this group can be given. Raise them for bigger VMs. |
| `storage` | `image` | Each VM's disk is a file in the working directory. It's simple and persistent with no extra setup. See [snapshot-based storage](/storage/overview) for faster clones. |
| `storage_size` | `25G` | The size of each VM's disk. |
| `persistent` | `true` | Keep each VM's disk when it's shut down or Slicer restarts, so it can be relaunched. |
| `network` | `brvm0` | VMs join a Linux bridge and get addresses from `192.168.137.0/24`. |
| `image` | `slicer-systemd` | Ubuntu 22.04 with systemd for your host's architecture. Ubuntu 24.04, Rocky Linux and Arch Linux are also [available](/reference/images). |
| `hypervisor` | `firecracker` | `cloud-hypervisor` and `qemu` are also supported. |
| `api` | `127.0.0.1:8080` | The API used by the `slicer` CLI, protected by a token in `/var/lib/slicer/auth/token`. |

Run `slicer new --help` to see all the options. Every field is covered in the [config reference](/reference/config).

## 2. Start the Slicer daemon

The daemon needs root to manage KVM and networking. It's the only command in this guide that needs `sudo`.

To try Slicer out, run the daemon in the foreground:

```bash
cd ~/slicer
sudo slicer up
```

`slicer up` loads `slicer.yaml` from the current directory. To use another file, pass its path i.e. `sudo slicer up ./other.yaml`. The OS image is pulled on the first start, which takes a few seconds.

Leave it running and open a new terminal for the next steps. Press `Control+C` to stop it. If you're connected over SSH, you could start it inside `tmux` or `screen` to keep it running after you disconnect.

**Or install it as a service from the start**

If you already know you want Slicer running permanently, skip the foreground run and install it as a systemd service instead. It'll start on boot and restart if it fails:

```bash
cd ~/slicer
sudo slicer service generate --install
```

This creates a service named `slicer` that loads `slicer.yaml`, with `~/slicer` as its working directory. Run it from the same directory where you created the config, because that's where Slicer keeps each VM's disk and its state.

Follow its logs with `journalctl -u slicer -f`. See [running in the background](/getting-started/daemon) for more options.

## 3. Launch a VM by name

Launch a VM called `dev`:

```bash
slicer vm add vm --name dev
```

```
VM created
  Hostname: vm-1
  Name:     dev
  Group:    vm
  Arch:     amd64
  IP:       192.168.137.2/24
  Specs:    2 vCPU, 4GiB RAM, GPUs: 0
  Persistent: true
  Duration: 2.944s
```

* `vm` is the host group to launch into.
* `--name dev` gives the VM a friendly name to use with every other command.

`slicer vm add` returns as soon as the VM is created, whilst it carries on booting in the background. Wait for it to be ready:

```bash
slicer ready dev
```

This returns once the guest agent inside the VM is up, usually in under a second. In scripts, you can combine both steps with `slicer vm add vm --name dev --wait`.

List your VMs:

```bash
slicer ls
```

```
AGE  NAME  HOSTNAME  STATUS   IP             RAM   CPU  PERSIST  TAGS
---  ----  --------  ------   --             ---   ---  -------  ----
0s   dev   vm-1      Running  192.168.137.2  4GiB  2    yes
```

Each VM also gets a generated hostname such as `vm-1`. Either will work with the CLI, but the name is easier to remember. Add `--tag` to attach your own metadata i.e. `--tag project=website`.

## 4. Work inside the VM

Open a shell:

```bash
slicer shell dev
```

<div class="terminal">
<div class="terminal-bar"><span></span><span></span><span></span><em>alex@host: ~/slicer</em></div>
<pre><span class="prompt">alex@host:~/slicer$</span> slicer shell dev
<span class="muted">Connecting to VM: vm-1
Connected! Press Ctrl+] to exit.</span>
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.1.90 x86_64)

* Image built by SlicerVM (2026)
* Website: https://slicervm.com
* Docs: https://docs.slicervm.com

<span class="prompt">ubuntu@vm-1:~$</span> uname -r &amp;&amp; uptime
6.1.90
 19:20:57 up 4 min,  0 users,  load average: 0.00, 0.00, 0.00
<span class="prompt">ubuntu@vm-1:~$</span></pre>
</div>

Type `exit` or press `Control+]` to leave the shell.

If you leave off the VM name, `slicer shell`, `slicer vm exec` and `slicer vm logs` use the most recently launched VM.

You'll be logged in as the `ubuntu` user with passwordless `sudo`. On the Rocky Linux and Arch images the user is `slicer`. The shell runs over Slicer's guest agent, so no SSH keys or network routes are needed.

Run a one-off command:

```bash
slicer vm exec dev -- "uname -a && df -h /"
```

```
Linux vm-1 6.1.90 #1 SMP PREEMPT_DYNAMIC Sun Sep 13 00:30:58 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        25G  756M   23G   4% /
```

Copy files in and out, and add `-r` for a folder:

```bash
slicer cp ./notes.txt dev:~/notes.txt
slicer cp -r ./project dev:~/project
slicer cp dev:~/notes.txt ./notes.txt
```

Install a web server:

```bash
slicer vm exec dev -- "sudo apt-get update -qq && sudo apt-get install -y nginx"
```

With bridge networking, each VM's IP is reachable directly from the Slicer host:

```bash
curl -s http://192.168.137.2 | head -n 4
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

You can also forward a port to `localhost`. That's useful when the VM is on an isolated network, or when you're [using Slicer from another machine](#optional-use-slicer-from-another-machine):

```bash
slicer vm forward dev -L 3000:127.0.0.1:80
```

Then open `http://127.0.0.1:3000`. The forward runs until you press `Control+C`. Slicer's API already uses port `8080` on the host, so pick a different local port.

## 5. Launch more, and clean up

Launch as many VMs as your host can hold. Each one can be smaller than the host group's `vcpu` and `ram_gb` but not bigger:

```bash
slicer vm add vm --name build --cpus 1 --ram-gb 2
```

```
VM created
  Hostname: vm-2
  Name:     build
  Group:    vm
  Arch:     amd64
  IP:       192.168.137.3/24
  Specs:    1 vCPU, 2GiB RAM, GPUs: 0
  Persistent: true
  Duration: 1.076s
```

Add `--rm` for a throwaway VM whose disk is discarded when it stops.

View a VM's boot log and then delete it:

```bash
slicer vm logs build | tail -n 3
```

```
[    0.698393] vmmeter[597]: 2026/10/12 19:21:55 Collecting metrics for: vm-2 on port: 34723
[    0.700387] configure-dns.sh[565]: + '[' -f /runner/userdata.sh ']'
[    0.701032] vmmeter[597]: 2026/10/12 19:21:55 VSOCK listening on port: 515
```

```bash
slicer vm delete build
```

```
Deleted vm-2 (group vm)
```

VMs are persistent by default. When you stop and start Slicer, `dev` will be relaunched with its disk intact along with any files and packages you added.

When Slicer stops, it shuts each VM down cleanly so nothing is lost. For throwaway sandboxes, you can generate the config with `slicer new --graceful-shutdown=false` so Slicer exits straight away instead.

## 6. Hand over to your coding agent

With the [agent skills](#give-your-agent-its-own-computer) installed, your agent uses the same commands you've just run. Some things to try next:

<div class="agent-prompt" markdown>
Copy this repo into my `dev` VM, build it there, and bring the binary back to `./bin`.
</div>

<div class="agent-prompt" markdown>
Install Docker in my `dev` VM and forward its socket, so I can run `docker build` locally against it.
</div>

<div class="agent-prompt" markdown>
Use Slicer to launch a new VM called `k3s`, install K3s in it, and merge the kubeconfig into `~/.kube/config`.
</div>

To run a coding agent itself inside a VM, see [Sandbox your agent in a microVM](/getting-started/agent-sandbox).

## Optional: SSH into VMs

You don't need SSH to work with Slicer. `slicer shell`, `exec`, `cp` and `forward` all go through Slicer's API and the guest agent inside each VM.

SSH is handy for tools that expect it, such as `scp`, `rsync` or VS Code's Remote SSH. VMs run an OpenSSH server. To add your keys, pass one of these to `slicer new`:

* `--find-ssh-keys` - add the public keys found in `~/.ssh/`
* `--github USERNAME` - import the keys from your GitHub profile
* `--ssh-key "ssh-ed25519 ..."` - add a key directly

They're written to the config under `ssh_keys` or `github_user`.

From the Slicer host, connect using the VM's IP:

```bash
ssh ubuntu@192.168.137.2
```

To reach VMs from another machine on your network, print the route commands with `slicer vm route ./slicer.yaml` and run them on that machine. Never run them on the Slicer host itself, as it will break its networking.

See [SSH access to VMs](/reference/ssh) for more.

## Optional: Use Slicer from another machine

The `slicer` CLI is a client for the daemon's REST API. The API binds to `127.0.0.1` by default, but if you expose it over HTTPS, you can drive Slicer from your laptop or anywhere else.

Copy the token from `/var/lib/slicer/auth/token` on the Slicer host, then point the CLI at the API:

```bash
export SLICER_URL=https://slicer.example.com
export SLICER_TOKEN_FILE=~/.slicer/token

slicer ls
slicer shell dev
```

Everything in this guide works the same way remotely, including `cp` and `forward`. See the [API reference](/reference/api#authentication) for exposing it safely with an inlets tunnel or Caddy.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No license found on start-up | Your license key is emailed to you when you start a trial or subscribe. Save it to `~/.slicer/LICENSE`. |
| `permission denied` on `/var/lib/slicer/auth/token` | Join the `slicer` group with `sudo usermod -aG slicer $USER`. Then log out and back in or run `newgrp slicer`. |
| `requested RAM ... exceeds host group limit` | `--cpus` and `--ram-gb` can't exceed the host group's `vcpu` and `ram_gb`. Raise them in `slicer.yaml` and restart Slicer. |
| `Bridge conflict: ... already in use` | A previous run didn't exit cleanly. Start with `sudo slicer up --force`. |
| Your LAN already uses `192.168.137.0/24` | Regenerate the config with a free range, i.e. `--cidr 192.168.140.0/24`. |
| `connection refused` from the CLI | Slicer isn't running. Check the terminal where you ran `slicer up`, or `systemctl status slicer`. |

## Next steps

* [Automate a project's workspace with `slicer workspace`](/getting-started/workspaces), Slicer's alternative to a Dockerfile and Compose.
* [Customise VMs with userdata](/tasks/userdata) to install packages on first boot.
* [Use snapshot-based storage](/storage/overview) with ZFS or devmapper for near-instant VM creation.
* Try an example, such as [running Docker](/examples/docker) or a [HA Kubernetes cluster](/examples/ha-k3s).
