# Persistent Linux VM

Slicer for Mac has two types of VMs.

1. A Persistent Linux VM named `slicer-1` - analogous to WSL2 - your *Linux twin* for macOS.
2. Additional persistent or ephemeral VMs ["sandboxes"](/mac/sandboxes.md) can be launched into the `sbox` host group.

Unlike most sandboxes that optimise for a narrow use-case, each VM is a full Linux Kernel with support for Docker, K3s, eBPF, coding agents, Go/Rust builds with systemd as the init.

Additionally, you can share your home folder or any other folder directly into the VM via VirtioFS.

A built-in guest agent can be used instead of SSH for faster, more efficient access:

- `slicer vm shell slicer-1`
- `slicer vm cp ...`
- `slicer vm forward ...`

SSH is pre-installed, and accessible via the VM's IP address, as shown on `slicer vm list`.

You can add your SSH keys to: `~/.ssh/authorized_keys`, or import them directly from GitHub:

```bash
slicer vm shell slicer-1

curl -sLS https://github.com/alexellis.keys > ~/.ssh/authorized_keys
```

## Architecture diagram

```text
                         +----------------------------+
                         |          slicer CLI        |
                         |   (vm shell / vm cp / API) |
                         +-------------+--------------+
                                       |
                                       v
      +--------------------------------+-----------------------------------+
      |              slicer-mac daemon on macOS                            |
      |  Reads `slicer-mac.yaml` and controls local microVMs               |
      +-----------------------+----------------------+---------------------+
                              |                      |
                              |                      |
                              v                      v
             +-----------------------------+    +----------------------------+
             | host_group: slicer          |    | host_group: sbox           |
             | Long-lived primary workload |    | Disposable / on-demand VMs |
             +--------------+--------------+    +-------------+--------------+
                            |                                |
                            v                                v
                     +-------------+                  +----------------+
                     |   slicer-1  |                  |    sbox-1      |
                     | main VM     |                  | sample sbox VM |
                     +-------------+                  +----------------+
```

Docker's socket is port-forwarded to your Mac as a Unix socket, so `docker` commands on the Mac talk directly to the VM. K3s exposes port 6443, so `kubectl` on your Mac can target the cluster running inside `slicer-1`.

## The VM lifecycle

It's important to shut down persistent VMs like `slicer-1` gracefully:

```bash
slicer vm shutdown slicer-1
slicer vm exec slicer-1 -- sudo shutdown -h 0
```

If your VM crashes or you kill slicer-mac without letting it shut down the VMs gracefully, you may need to check the disk image. See [Check or repair a VM disk image](/mac/troubleshooting/#check-or-repair-a-vm-disk-image).

If you ever want to "reset" your `slicer-1` VM, you can delete it and then relaunch it.

First shut down slicer-mac.

Then run `rm -rf ~/slicer-mac/slicer-1.img`

Then restart slicer-mac, and you'll get the VM re-created.

## Folder sharing

Folders can be shared directly into any Slicer VM by specifying paths in the slicer-mac.yaml config file or via an API request.

Most of the time copying folders between the host and guest, will be fast enough and more convenient: `slicer cp -r ./source vm:~/destination`.

See [Folder sharing](/mac/folder-sharing) for setup details.

## Forward Docker

Install Docker in the VM if it's not already present:

```bash
curl -sLS https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
sudo systemctl enable docker --now
```

Forward Docker's socket to your Mac so `docker` commands work natively on the host:

```bash
slicer vm forward slicer-1 \
  -L ~/.slicer/docker.sock:/var/run/docker.sock
```

Then on your Mac:

```bash
export DOCKER_HOST=unix://$HOME/.slicer/docker.sock

docker ps
```

Add the `DOCKER_HOST` export to your `~/.zshrc` or `~/.bashrc` to make it permanent.

## Forward K3s

Install K3s tooling in the guest first if needed:

```bash
arkade get k3sup kubectl
k3sup install --local
```

Forward K3s port 6443 so `kubectl` on your Mac can reach the cluster:

```bash
slicer vm forward slicer-1 \
  -L 6443:127.0.0.1:6443
```

Then point kubectl at it:

```bash
slicer vm cp slicer-1:/etc/rancher/k3s/k3s.yaml ~/.kube/slicer-k3s-config
export KUBECONFIG=$HOME/.kube/slicer-k3s-config

kubectl get nodes
```

With K3s running inside Slicer, you can test controllers locally, validate Helm charts with a real install, or try RBAC changes without touching a shared cluster.

## Next steps

- [Sandboxes](/mac/sandboxes) - spin up ephemeral VMs for AI agents and automation
- [Copy files to/from a VM](/tasks/copy-files) - use `slicer vm cp` to move files in and out
- [Execute commands in a VM](/tasks/execute-commands) - run commands remotely with `slicer vm exec`
