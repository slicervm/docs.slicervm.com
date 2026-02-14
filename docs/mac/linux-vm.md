# Linux VM

Your main VM is `slicer-1` in the `slicer` host group. It's a persistent arm64 Linux environment that boots with the daemon and has your Mac home folder shared in via VirtioFS. This is where you do your day-to-day Linux work - running Docker, K3s, coding agents, Go/Rust builds, and anything else you'd do on a Linux workstation.

SSH is usually not required for normal workflows. Most tasks are faster with the Slicer CLI:

- `slicer vm shell slicer-1`
- `slicer vm cp ...`
- `slicer vm port-forward ...`

Use SSH only when you need direct shell access outside this interface. You can add keys directly in the guest by writing to `~/.ssh/authorized_keys`:

```bash
curl -sLS https://github.com/alexellis.keys > ~/.ssh/authorized_keys
```

## Folder sharing

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
slicer vm port-forward slicer-1 \
  ~/.slicer/docker.sock:/var/run/docker.sock
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
slicer vm port-forward slicer-1 \
  6443:127.0.0.1:6443
```

Then point kubectl at it:

```bash
slicer vm cp slicer-1:/etc/rancher/k3s/k3s.yaml ~/.kube/slicer-k3s-config
export KUBECONFIG=$HOME/.kube/slicer-k3s-config

kubectl get nodes
```

With K3s running inside Slicer, you can test controllers locally, validate Helm charts with a real install, or try RBAC changes without touching a shared cluster.

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

## Next steps

- [Sandboxes](/mac/launch-sandboxes) - spin up ephemeral VMs for AI agents and automation
- [Copy files to/from a VM](/tasks/copy-files) - use `slicer vm cp` to move files in and out
- [Execute commands in a VM](/tasks/execute-commands) - run commands remotely with `slicer vm exec`
