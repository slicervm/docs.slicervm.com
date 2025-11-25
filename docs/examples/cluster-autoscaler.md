# Cluster Autoscaler with Slicer

The Slicer Cluster Autoscaler enables automatic scaling of Kubernetes nodes on bare metal using Firecracker microVMs. This example demonstrates how to set up a HA K3s control plane that can automatically scale worker nodes based on Pod scheduling demands.

## Architecture Overview

* **Control Plane**: K3s cluster running on one Slicer host group (3 nodes recommended)
* **Worker Nodes**: Separate Slicer host group starting with zero nodes, scaled automatically by the cluster autoscaler
* **Autoscaler**: Deployed in-cluster using Helm, managing both the K3s API and Slicer APIs to provision new worker nodes

## Learn More

Watch a video walkthrough on our [YouTube channel](https://www.youtube.com/watch?v=MHXvhKb6PpA&t).

<iframe width="560" height="315" src="https://www.youtube.com/embed/MHXvhKb6PpA?si=AerZGyu30zm-cRDF" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Step 1: Setup Kubernetes cluster

Create and deploy a 3-node K3s control plane. This will serve as the foundation for your autoscaling cluster. For detailed instructions on Highly Available K3s setup with Slicer, see the [HA K3s example](/examples/ha-k3s).

Use `slicer new` to generate a configuration file for the control plane nodes. We specify 3 nodes for high availability:

```bash
# Create control plane configuration
slicer new k3s-cp \
  --cpu 2 \
  --ram 4 \
  --count=3 \
  --cidr 192.168.137.0/24 \
  --api-port=8080 \
  --api-bind="0.0.0.0" \
  > k3s-cp.yaml

# Deploy control plane
sudo -E slicer up ./k3s-cp.yaml
sudo -E slicer vm list --json > devices.json
```

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

## Step 2: Setup Worker Node Host Group

Create a separate Slicer deployment for autoscaled worker nodes. This host group can run on the same virtualization server as the control plane (using a different CIDR) or on a separate host.

```bash
# Create worker node host group (starting with 0 nodes)
slicer new k3s-agents \
  --cpu 2 \
  --ram 4 \
  --cidr 192.168.138.0/24 \
  --count=0 \
  --api-port=8081 \
  --ssh-port=2223 \
  > k3s-agents.yaml

# Deploy worker host group
sudo -E slicer up ./k3s-agents.yaml
```

This host group starts with zero nodes (`count=0`). The cluster autoscaler will call the Slicer API to dynamically add nodes to this group based on scheduling demands.

Retrieve the API token from the worker node Slicer instance. This token will be used by the cluster autoscaler to authenticate with the Slicer API when provisioning new worker nodes:

```bash
# On the worker node host
sudo cat /var/lib/slicer/auth/token
```

You can repeat this process on multiple hosts to create additional worker node groups if needed.

## Step 3: Configure Networking

Configure network routing based on your deployment setup:

If running on different hosts (recommended):

- On the control plane host: Add routes to reach worker node networks
- On worker node hosts: Add routes to reach the control plane network

```bash
# On control plane host (only if hosts are different)
sudo ip route add 192.168.138.0/24 via <worker-host-ip>

# On worker node host (only if hosts are different)
sudo ip route add 192.168.137.0/24 via <control-plane-host-ip>
```

## Step 4: Configure Cluster Autoscaler

Create the cloud configuration file that tells the cluster autoscaler how to connect to your K3s cluster and Slicer APIs. This INI-format file defines the node groups and their scaling parameters:

```bash
cat > cloud-config.ini <<EOF
[global]
k3s-url=https://192.168.137.2:6443
k3s-token=$(cat ~/k3s-join-token.txt)
default-min-size=0
default-max-size=10

[nodegroup "k3s-agents"]
slicer-url=http://192.168.138.1:8081
slicer-token=<SLICER_API_TOKEN>
EOF
```

Replace `<YOUR_SLICER_TOKEN>` with the actual token retrieved from Step 2. The `k3s-url` should point to your control plane, and the `slicer-url` should point to your worker node host group's API endpoint.

Note that the `nodegroup` name be the same as the Slicer host group name.

Overview of all available configuration options:

| Key | Description | Required | Default |
|-----|-------------|----------|---------|
| `global/k3s-url` | K3s control plane API server URL | Yes | - |
| `global/k3s-token` | K3s join token for new nodes | Yes | - |
| `global/default-min-size` | Default minimum nodes per group | No | 1 |
| `global/default-max-size` | Default maximum nodes per group | No | 8 |
| `nodegroup/slicer-url` | Slicer API server URL | Yes | - |
| `nodegroup/slicer-token` | Slicer API authentication token | Yes | - |
| `nodegroup/min-size` | Group-specific minimum size | No | global default |
| `nodegroup/max-size` | Group-specific maximum size | No | global default |

## Step 5: Deploy Cluster Autoscaler

Deploy the cluster autoscaler using Helm. First, create a Kubernetes secret containing the cloud configuration file:

```bash
kubectl create secret generic cluster-autoscaler-cloud-config \
  --from-file=cloud-config=cloud-config.ini \
  -n kube-system
```

Create a `values.yaml` file to configure the cluster autoscaler Helm chart. This configuration specifies the Slicer-compatible autoscaler image, mounts the cloud config secret, and sets appropriate scaling parameters:

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
helm upgrade --install slicer-autoscaler autoscaler/cluster-autoscaler \
  --namespace=kube-system \
  --values=values.yaml
```

## Testing the Autoscaler

Test the autoscaler by creating workloads that cannot be scheduled on the existing control plane nodes. You can either scale a deployment beyond current capacity or create pods with specific scheduling constraints that prevent them from running on control plane nodes.

Monitor the autoscaler:

```bash
# Watch autoscaler logs
kubectl logs -f deployment/slicer-autoscaler -n kube-system

# Watch nodes being added
kubectl get nodes -w

# Check Slicer API for new VMs
curl -H "Authorization: Bearer <token>" http://192.168.138.1:8081/nodes
```

The autoscaler will detect unschedulable pods and automatically provision new worker nodes through the Slicer API. Once workloads are removed or reduced, it will scale down unneeded nodes after the configured delay period.
