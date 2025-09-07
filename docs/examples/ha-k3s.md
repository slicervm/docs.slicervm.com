# Highly Available (HA) Kubernetes with K3s

The following example sets up a 3x Node Kubernetes cluster using K3s. As an optional step, you can set up a Load Balancer running on the Slicer host to distribute traffic across the nodes for the API server and HTTP/HTTPS.

If you would like to try out a smaller cluster first, you can simply change the `count` from `3` to `1` when saving the file below.

Create `k3s-ha.yaml`:

```yaml
config:
  host_groups:
  - name: k3s
    storage: image
    storage_size: 25G
    count: 3
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brk3s0
      tap_prefix: k3stap
      gateway: 192.168.137.1/24

  github_user: alexellis

  api:
    port: 8080
    bind_address: "127.0.0.1:"

  kernel_image: "ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-latest"
  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker
```

The IP addresses of the VMs will be as follows:

* `192.168.137.2`
* `192.168.137.3`
* `192.168.137.4`

## Setup Kubernetes with K3sup Pro

Download K3sup Pro:

```bash
PRO=true curl -sSL https://get.k3sup.dev | sudo sh
```

If you want to leave off `sudo`, then just move the `k3sup` binary into your `$PATH` variable manually.

Next, on the host where slicer is running, get the devices file from Slicer's API:

```bash
curl -sLS http://127.0.0.1:8080/nodes > devices.json
```

Copy devices.json back to your workstation.

On your workstation, add any routes that are specified so you can access the VMs on their own network.

Check the options like disabling Traefik, so that you can install Ingress Nginx or Istio instead.

```bash
k3sup plan --help

k3sup plan --traefik=false --user ubuntu
```

This will generate a plan.yaml file, you can review and edit it.

Next, run `k3sup apply`.

This will install the first server, then server 2 and 3 in parallel.

Finally run:

```bash
mkdir -p ~/.kube
cp ~/.kube/config ~/.kube/config.bak || true

k3sup get-config \
 --local-path ~/.kube/config \
 --merge \
 --context slicer-k3s-ha
```

Then you can run `kubectx slicer-k3s-ha`, and start using kubectl.

Your cluster is running in HA mode.

## Create a HA LoadBalancer for the VMs

If you would like to create a load balancer for the microVMs, you can do so using the mixctl add-on.

```
arkade get mixctl
```

Create a config named `k3s.yaml`:

```yaml
version: 0.1

rules:
- name: k3s-api
  from: 127.0.0.1:6443
  to:
    - 192.168.137.2:6443
    - 192.168.137.3:6443
    - 192.168.137.4:6443

- name: k3s-http
  from: 127.0.0.1:80
  to:
    - 192.168.137.2:80
    - 192.168.137.3:80
    - 192.168.137.4:80

- name: k3s-tls
  from: 127.0.0.1:443
  to:
    - 192.168.137.2:443
    - 192.168.137.3:443
    - 192.168.137.4:443
```

Then run `mixctl ./k3s.yaml`

Finally, revisit your plan so each server obtains a TLS certificate for the Kubernetes API server for the IP address of the Slicer host.

So if the Slicer host were 192.168.1.100:

```bash
k3sup plan --tls-san 192.168.1.100 \
  --update

k3sup apply
```

Then edit your `~/.kube/config` file and replace `192.168.137.2:6443` with `192.168.1.100:6443`.

Now every time you run `kubectl`, you'll see mixctl balance traffic across all three servers.

