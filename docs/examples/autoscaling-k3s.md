# Autoscaling Kubernetes (k3s) with Cluster Autoscaler

The Cluster Autoscaler is a controller offered by the Kubernetes community to add and remove nodes to a a Kubernetes cluster based upon demand.

When run in the cloud such as AWS, Azure, GCP, etc, the autoscaler provisions nodes using the cloud provider's VM primitive, so EC2 for AWS, Google Compute Engine for GCP, etc.

With Slicer it launches Firecracker microVMs on one or more Slicer hosts. So you can build massive clusters at home or in your own datacentre.

* Adds new Kubernetes nodes via Slicer's REST API
* Removes nodes when they are no longer needed
* Acts in a similar way to spot instances (ephemeral nodes)

## Conceptual Overview

**Control Plane**

The Control Plane nodes will be setup statically via YAML configuration on one Slicer host. The default is 3, but you could increase that if you wished.

The Cluster Autoscaler will be deployed in-cluster using Helm, with a TOML file that defines the node groups and their scaling parameters, and which Slicer API and Token URL to use.

**Workers / Agents**

The workers or agents will be provisioned as required by the Cluster Autoscaler, pointing at one or most hosts running Slicer's REST API.

## Demo: Watch a video walkthrough

If you'd like to see how it works, you can watch a [walkthrough video on YouTube](https://www.youtube.com/watch?v=MHXvhKb6PpA).

<iframe width="560" height="315" src="https://www.youtube.com/embed/MHXvhKb6PpA?si=AerZGyu30zm-cRDF" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Step 1: Setup Kubernetes cluster

On the first Slicer host, the one that will run the control plane run the following steps.

Create and deploy a 3-node K3s control plane. This will serve as the foundation for your autoscaling cluster. For detailed instructions on Highly Available K3s setup with Slicer, see the [HA K3s example](/examples/ha-k3s).

Use `slicer new` to generate a configuration file for the control plane nodes.

```bash
# Create control plane configuration
slicer new k3s-cp \
  --cpu 2 \
  --ram 4 \
  --count=3 \
  --cidr 192.168.137.0/24 \
  > k3s-cp.yaml
```

The values above are suggestions that we've tested, but you can customise the specifications to your own needs.

* `--cpu` - the number of virtual CPUs to allocate to each control plane VM
* `--ram` - the amount of RAM in GB to allocate to each control plane VM
* `--count` - the number of control plane VMs to create
* `--cidr` - the CIDR block to use for the control plane network (best not to change this until you've run the whole tutorial successfully)

Then start up the control plane nodes:

```bash
# Deploy control plane
sudo -E slicer up ./k3s-cp.yaml
sudo -E slicer vm list --json > devices.json
```
## Step 1.1: Install K3s on the control plane nodes

Move over to your workstation.

Copy the devices.json file to your workstation.

Add the route that you were given when starting up Slicer to your machine.

Install K3s across all control plane nodes using K3sup Pro. K3sup Pro automates the installation process, setting up the first server and then joining the remaining servers in parallel:

```bash
# Download K3sup Pro (included with Slicer)
curl -sSL https://get.k3sup.dev | PRO=true sudo -E sh
k3sup-pro activate

# Create K3s cluster
k3sup-pro plan --user ubuntu ./devices.json
k3sup-pro apply

# Get kubeconfig and join token
k3sup-pro get-config --local-path ~/k3s-cp-kubeconfig
k3sup-pro node-token --user ubuntu --host 192.168.137.2 > ~/k3s-join-token.txt
```

The kubeconfig is required so that you can use kubectl to manage the cluster. There are additional flags if you want to merge this into your existing kubeconfig file under a new context name such as:

```bash
k3sup-pro get-config \
  --local-path ~/.kube/config \
  --merge \
  --context slicer-k3s-cp
```

The `k3sup-pro node-token` command fetches the token that K3s will use to add new agents/workers into the cluster, and will be consumed by the Cluster Autoscaler when it is deployed.

## Step 2: Setup Worker Node Host Group

Create a separate Slicer instance for autoscaled worker nodes. This could be done on the same machine as the control plane, or a different one.

Note that the CIDR block must be different for each Slicer instance.

```bash
# Create worker node host group (starting with 0 nodes)
slicer new k3s-agents-1 \
  --cpu 2 \
  --ram 4 \
  --cidr 192.168.138.0/24 \
  --count=0 \
  --tap-prefix="k3sa1"
  --api-bind="0.0.0.0" \
  --api-port=8081 \
  --ssh-port=2223 \
  > k3s-agents-1.yaml

# Deploy worker host group
sudo -E slicer up ./k3s-agents-1.yaml
```

This host group starts with zero nodes (`count=0`). The Cluster Autoscaler will call the Slicer API to dynamically add nodes to this group based on scheduling demands.

Retrieve the API token from this Slicer instance. This token will be used by the Cluster Autoscaler to authenticate with the Slicer API when provisioning new worker nodes:

```bash
# On the worker node host
sudo cat /var/lib/slicer/auth/token
```

Save the value on your workstation as `~/slicer-token-1.txt`.

You can repeat this process on multiple hosts to create additional worker node groups if needed.

## Step 2.1: Enroll other machines

If you have additional machines that could run Slicer and provide Kubernetes nodes, you can add them by following the instructions above.

This could be an Arm64 machine (Ampere, Raspberry Pi, etc) or an x86_64 (Intel, AMD, etc) machine.

For each, make sure you change the CIDR block, and the tap-prefix.

I.e. to add `k3s-agents-2`:

```bash
slicer new k3s-agents-2 \
  --cpu 2 \
  --ram 4 \
  --cidr 192.168.139.0/24 \
  --count=0 \
  --tap-prefix="k3sa2"
  --api-bind="0.0.0.0" \
  --api-port=8082 \
  --ssh-port=2224 \
  > k3s-agents-2.yaml
```

Remember to save each Slicer API token to your workstation as `~/slicer-token-2.txt`, `~/slicer-token-3.txt`, etc.

## Step 3: Configure Networking

**Your workstation**

On your workstation, you can get away with only adding a route to the Control Plane Slicer instance.

**On each worker host**

Each worker host must have a route to:

* The Control Plane Slicer instance
* Every other worker node Slicer instance (other than its own)

## Step 4: Configure Cluster Autoscaler

Create the cloud configuration file that tells the Cluster Autoscaler how to connect to your K3s cluster and Slicer APIs.

This TOML file defines the node groups and their scaling parameters:

```bash
cat > cloud-config.ini <<EOF
[global]
k3s-url=https://192.168.137.2:6443
k3s-token=$(cat ~/k3s-join-token.txt)
default-min-size=0
default-max-size=10

[nodegroup "k3s-agents-1"]
slicer-url=http://192.168.138.1:8081
slicer-token=$(cat ~/slicer-token-1.txt)
EOF
```

If you were to have had two Slicer instances running worker nodes, the config would look like this:

```bash
cat > cloud-config.ini <<EOF
[global]
k3s-url=https://192.168.137.2:6443
k3s-token=$(cat ~/k3s-join-token.txt)
default-min-size=0
default-max-size=10

[nodegroup "k3s-agents-1"]
slicer-url=http://192.168.138.1:8081
slicer-token=$(cat ~/slicer-token-1.txt)

[nodegroup "k3s-agents-2"]
slicer-url=http://192.168.139.1:8082
slicer-token=$(cat ~/slicer-token-2.txt)
EOF
```

* The `nodegroup` name comes from the the hostgroup name of the Slicer instance
* The `slicer-url` is the API URL of the Slicer instance
* The `slicer-token` is the API token of the Slicer instance
* The `k3s-url` is the API URL of the K3s control plane
* The `k3s-token` is the join token for the K3s control plane
* The `default-min-size` is the default minimum number of nodes to scale to
* The `default-max-size` is the default maximum number of nodes to scale to

Overview of all available configuration options:

| Key                        | Description | Required | Default |
|----------------------------|-------------|----------|---------|
| `global`                   | Global configuration options | No | - |
| `global/k3s-url`           | K3s control plane API server URL | Yes | - |
| `global/k3s-token`         | K3s join token for new nodes | Yes | - |
| `global/default-min-size`  | Default minimum nodes per group | No | 1 |
| `global/default-max-size`  | Default maximum nodes per group | No | 8 |
| `nodegroup/slicer-url`     | Slicer API server URL | Yes | - |
| `nodegroup/slicer-token`   | Slicer API authentication token | Yes | - |
| `nodegroup/min-size`       | Group-specific minimum size | No | global default |
| `nodegroup/max-size`       | Group-specific maximum size | No | global default |

## Step 5: Deploy Cluster Autoscaler

Deploy the Cluster Autoscaler using Helm.

If you don't have Helm, you can install it via [arkade](https://arkade.dev) with: `arkade get helm`.

First, create a Kubernetes secret containing the cloud configuration file:

```bash
kubectl create secret generic cluster-autoscaler-cloud-config \
  --from-file=cloud-config=cloud-config.ini \
  -n kube-system
```

Create a `values-slicer.yaml` file to configure the Cluster Autoscaler Helm chart. This configuration specifies the Slicer-compatible autoscaler image, mounts the cloud config secret, and sets appropriate scaling parameters:

```yaml
image:
  repository: docker.io/openfaasltd/cluster-autoscaler-slicer
  tag: latest

cloudProvider: slicer

autoDiscovery:
  clusterName: k3s-slicer

extraVolumeSecrets:
  cluster-autoscaler-cloud-config:
    name: cluster-autoscaler-cloud-config
    mountPath: /etc/slicer/
    items:
      - key: cloud-config
        path: cloud-config

extraArgs:
  cloud-config: /etc/slicer/cloud-config
  logtostderr: true
  stderrthreshold: info
  v: 4
  scale-down-enabled: true
  scale-down-delay-after-add: "30s"
  scale-down-unneeded-time: "30s"
  expendable-pods-priority-cutoff: -10
```

Deploy the autoscaler using the official Kubernetes autoscaler Helm chart:

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm upgrade --install \
  slicer-autoscaler autoscaler/cluster-autoscaler \
  --namespace=kube-system \
  --values=./values-slicer.yaml
```

## Test the Cluster Autoscaler

The autoscaler works by detecting unschedulable Pods and automatically provisioning new Worker Nodes through Slicer's REST API. Once workloads are removed or reduced, it will scale down unneeded nodes after the configured cooldown period.

The capacity of the Control Plane will mean that you can already launch a large number of Pods without needing any scaling.

Here's how you can force the Cluster Autoscaler to scale:

* Fill up the Control Plane nodes with Pods
* Use taints, tolerations, nodeselector labels, or other affinity rules to prevent Pods from running on the Control Plane nodes

You can watch the logs of the autoscaler to understand what it's doing:

```bash
# Watch autoscaler logs
kubectl logs -f deployment/slicer-autoscaler \
  -n kube-system
```

In another terminal, ideally a split tmux pane set up the below:

```bash
# Pane 1 - watch nodes being added
kubectl get nodes --watch --output wide

# Pane 2 - watch which Pods are unschedulable/Not Ready - that means they're waiting for a node to be available
kubectl get pods --watch --output wide --all-namespaces
```

Keep another eye out on Slicer's output on the worker node host. You should see VMs being booted up and / or shutting down. 

