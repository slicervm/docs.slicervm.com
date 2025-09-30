# OpenFaaS Pro and CE in Slicer

Boot up a pre-configured microVM with OpenFaaS Pro or CE setup along with `faas-cli`, `helm`, and `stern` for log tailing.

For either edition, the URL for the gateway is set up for you for in the default user's `.bashrc` file:

```bash
export OPENFAAS_URL=http://127.0.0.1:31112
```

You can use that address directly on the VM during an SSH session, but you'll also be able to access the gateway from other machines by using the VM's IP i.e. `http://192.168.137.2:31112`.

Get instructions to fetch the password via `arkade info openfaas`.

For SSH access, customise the `github_user` field, or set the keys as a list via `ssh_keys`. Learn more about [SSH in Slicer](/reference/ssh).

A multi-node setup is possible, but in that case, it's better to set up K3s using K3sup Pro from outside the VMs, and to follow the [HA K3s example](/examples/ha-k3s) followed by the [OpenFaaS for Kubernetes instructions](https://docs.openfaas.com/deployment/).

## OpenFaaS Pro

Just copy your license key into the YAML file below and run `sudo slicer up -f openfaas-pro.yaml`.

Create `openfaas-pro.yaml` with the following content:

```yaml
config:

  host_groups:
  - name: openfaas-pro
    userdata: |
              export LICENSE=""
              export HOME=/home/ubuntu
              export USER=ubuntu

              cd /home/ubuntu/

              (
              arkade get kubectl kubectx helm faas-cli k3sup stern --path /usr/local/bin
              chown $USER /usr/local/bin/*

              mkdir -p .kube
              mkdir -p .openfaas

              echo -n $LICENSE > ./.openfaas/LICENSE
              )

              (
              k3sup install --local
              mv ./kubeconfig ./.kube/config
              chown $USER .kube/config
              )

              (
              kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

              kubectl create secret generic \
                -n openfaas \
                openfaas-license \
                --from-file license=$HOME/.openfaas/LICENSE

              helm repo add openfaas https://openfaas.github.io/faas-netes/
              helm repo update && \
                helm upgrade --install openfaas \
                --install openfaas/openfaas \
                --namespace openfaas \
                -f https://raw.githubusercontent.com/openfaas/faas-netes/refs/heads/master/chart/openfaas/values-pro.yaml

              chown -R $USER $HOME

              echo "export OPENFAAS_URL=http://127.0.0.1:31112" >> $HOME/.bashrc

              )

    storage: image
    storage_size: 25G
    count: 1
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24
      addresses:
      - 192.168.137.2/24
  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  api:
    port: 8080
    bind_address: "127.0.0.1:"
    auth:
      enabled: true

  ssh:
    port: 2222
    bind_address: "0.0.0.0:"

  hypervisor: firecracker
```

Edit `export LICENSE=""`

Then start it up:

```bash
sudo -E slicer up -f openfaas-pro.yaml
```

You can login with:

```bash
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

Deploy a function:

```bash
faas-cli store deploy figlet


Describe, then invoke the function:

```bash
faas-cli describe figlet
faas-cli invoke <<< "SlicerVM.com"
```

## OpenFaaS CE

OpenFaaS CE is licensed for personal, non-commercial use or a single 60 day commercial trial per company.

The above userdata script can be re-used, with a few options removed.

Create `openfaas-ce.yaml` with the following content:

```yaml
config:

  host_groups:
  - name: openfaas-ce
    userdata: |
              export HOME=/home/ubuntu
              export USER=ubuntu

              cd /home/ubuntu/

              (
              arkade get kubectl kubectx helm faas-cli k3sup stern --path /usr/local/bin
              chown $USER /usr/local/bin/*

              mkdir -p .kube
              mkdir -p .openfaas
              )

              (
              k3sup install --local
              mv ./kubeconfig ./.kube/config
              chown $USER .kube/config
              )

              (
              kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

              helm repo add openfaas https://openfaas.github.io/faas-netes/
              helm repo update && \
                helm upgrade --install openfaas \
                --install openfaas/openfaas \
                --namespace openfaas \

              chown -R $USER $HOME

              echo "export OPENFAAS_URL=http://127.0.0.1:31112" >> $HOME/.bashrc

              )

    storage: image
    storage_size: 25G
    count: 1
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24
      addresses:
      - 192.168.137.2/24
  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  api:
    port: 8080
    bind_address: "127.0.0.1:"
    auth:
      enabled: true

  ssh:
    port: 2222
    bind_address: "0.0.0.0:"

  hypervisor: firecracker
```
