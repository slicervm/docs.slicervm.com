# Multiple machine Kubernetes

In the [HA Kubernetes](/examples/ha-k3s.md) and [Large scale Kubernetes](/examples/large-scale-k3s.md) examples, we showed a single Kubernetes cluster running on a single Slicer host.

In this example, we'll show you spread Kubernetes clusters over multiple machines, each running Slicer.

1. You want a multi-arch Kubernetes cluster - with Slicer on an x86_64 and Arm64 machine
2. You want a much larger cluster than you can fit on any single machine

The basic idea is that you run Slicer on two or more machines, which manages its own set of VMs. These are put on distinct subnets to avoid IP address conflicts, then a rule is added to the routing table to enable inter-connectivity. This isn't limited to just Kubernetes, you could run any network-based product or service you like across multiple machines with this technique. 

## Install Slicer on each machine

First, install Slicer on each machine, following the [installation instructions](/getting-started/installation.md).

## Create a YAML file for each machine

You need to dedicate one subnet per machine, so that there are no IP address conflicts.

Then, you should ideally change the `name:` prefix of the host group, so that the machines have unique names.

The first machine could use:

```yaml
config:
  host_groups:
  - name: k3s-a
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

Then the second would have a different `name:` prefix, and a different network section.

```yaml
config:
  host_groups:
  - name: k3s-b
    network:
      bridge: brk3s0
      tap_prefix: k3stap
      gateway: 192.168.138.1/24
```

(Note the above is a partial snippet to show the differences only.)

## Start Slicer on each machine

On each respective machine run:

```bash
sudo -E slicer up ./k3s-a.yaml
```

```bash
sudo -E slicer up ./k3s-b.yaml
```

## Enable routing between the machines and VMs

Now, you need to be careful here.

Copy and paste the routing commands printed upon start-up.

1. On machine A, run the command from machine B
2. On machine B, run the command from machine A
3. On your workstation, which needs to access all of the VMs, run both commands

## Install Kubernetes with K3sup Pro

On each host run the following:

```bash
curl -SLsf http://127.0.0.1:8080/nodes > devices-a.json

curl -SLsf http://127.0.0.1:8080/nodes > devices-b.json
```

If you enabled authentication for the API, include the `-H "Authorization Bearer TOKEN"` header as printed out when Slicer started up.

Copy the two JSON files to your own workstation.

Switch over to a terminal on your own workstation.

[K3sup Pro](https://k3sup.dev) can support multiple devices.json files, the first file will be taken for the servers, the rest will be used as agents and installed in parallel.

```bash
PRO=1 curl -sSL https://get.k3sup.dev | sudo sh
```

```bash
k3sup-pro plan --user ubuntu \
  --devices ./devices-a.json \
  --devices ./devices-b.json \
  --servers 3 \
  --traefik=false
```

After the plan.yaml file is generated, you can run `k3sup-pro apply` to setup the cluster.

```bash
k3sup-pro apply
```

After a few moments, you can get the KUBECONFIG and merge it into your existing kubeconfig file:

```bash
mkdir -p ~/.kube
cp ~/.kube/config ~/.kube/config.bak || true

k3sup-pro get-config \
    --local-path ~/.kube/config \
    --merge \
    --context slicer-multi
```

Then you can run `kubectx slicer-multi` to switch to your new cluster and explore it with kubectl.

If your two machines were different kinds of architecture, explore the labels applied with:

```bash
kubectl get nodes --show-labels
```

Then you can deploy a Pod with a nodeSelector, or affinity/anti-affinity rule to control where your workloads run.

For a number of quick Kubernetes applications to try out, see `arkade install --help`.

The OpenFaaS CE app is quick to install and has a number of samples you can play with.
