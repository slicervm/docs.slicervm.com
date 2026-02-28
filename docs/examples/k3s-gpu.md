# Kubernetes with GPUs

In this example, we'll adapt elements of the [HA Kubernetes](/examples/ha-k3s) example and the [GPU Ollama](/examples/gpu-ollama) setup to work together, so you can launch Pods with GPU acceleration.

## Set up a config

Create the `k3s-gpu.yaml` file as below:

```yaml
config:
  host_groups:
  - name: gpu
    storage: image
    storage_size: 30G
    count: 1
    vcpu: 4
    ram_gb: 16
    gpu_count: 1
    network:
      bridge: brgpu0
      tap_prefix: gputap
      gateway: 192.168.139.1/24

  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd-ch:5.10.240-x86_64-latest"

  hypervisor: cloud-hypervisor
```

Feel free to customise the vCPU, RAM, and disk sizes.

Boot up the VM:

```bash
sudo slicer up ./k3s-gpu.yaml
```

Now, run the route commands so you can SSH into the host from your workstation.

Next, log into each VM via SSH:

```bash
ssh ubuntu@192.168.139.2
```

Next, install K3s using K3sup Pro or K3sup CE.

For CE:

```bash
k3sup install --host 192.168.139.2 --user ubuntu
```

You'll get a kubeconfig returned, run the commands to export it so kubectl uses it.

Install the Nvidia driver:

```bash
curl -SLsO https://raw.githubusercontent.com/self-actuated/nvidia-run/refs/heads/master/setup-nvidia-run.sh
chmod +x ./setup-nvidia-run.sh
sudo bash ./setup-nvidia-run.sh
```

Install the [Nvidia Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) using the official instructions. Use the instructions for "apt: Ubuntu, Debian".

Confirm that the nvidia container runtime has been found by K3s:

```bash
sudo grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
```

Apply a new runtime class for Nvidia:

```bash
cat > nvidia-runtime.yaml <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
EOF

kubectl create -f nvidia-runtime.yaml
```

Run a test Pod to show the output from nvidia-smi:

```bash
cat > nvidia-smi-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-smi
spec:
  runtimeClassName: nvidia
  restartPolicy: OnFailure
  containers:
    - name: nvidia-smi
      image: nvidia/cuda:12.1.0-base-ubuntu22.04
      command: ['sh', '-c', "nvidia-smi"]
EOF

kubectl create -f nvidia-smi-pod.yaml
```

For the time-being, this Pod uses only the `runtimeClassName` to request a GPU. Adding the usual `limits` section as below, does not work at present, and may require additional configuration in K3s or containerd:

```diff
+     resources:
+       limits:
+           nvidia.com/gpu: "1"
```

Fetch the logs:

```bash
kubectl logs nvidia-smi
```

```bash
$ kubectl logs pod/nvidia-smi
Mon Sep  1 15:04:11 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.76.05              Driver Version: 580.76.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        Off |   00000000:00:07.0 Off |                  N/A |
| 30%   46C    P0            110W /  350W |       0MiB /  24576MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## Enable Device Plugin

Install Nvidia's [Device Plugin](https://github.com/NVIDIA/k8s-device-plugin) for Kubernetes:

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
```

Patch it so it works with K3s:

```bash
# add runtimeClassName: nvidia to the DS pod spec
kubectl -n kube-system patch ds nvidia-device-plugin-daemonset \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/runtimeClassName","value":"nvidia"}]'

kubectl -n kube-system rollout status ds/nvidia-device-plugin-daemonset -n kube-system
```

Then run the Pod from earlier, but with the `limits` in place:

```bash
cat > nvidia-smi-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-smi
spec:
  runtimeClassName: nvidia
  restartPolicy: OnFailure
  containers:
    - name: nvidia-smi
      image: nvidia/cuda:12.1.0-base-ubuntu22.04
      command: ['sh', '-c', "nvidia-smi"]
      resources:
        limits:
            nvidia.com/gpu: "1"
EOF

kubectl create -f nvidia-smi-pod.yaml
```
