# OpenFaaS Pro and CE in Slicer

Boot up a pre-configured microVM with OpenFaaS Pro or CE setup along with `faas-cli`, `helm`, and `stern` for log tailing.

For either edition, the URL for the gateway is set up for you for in the default user's `.bashrc` file:

```bash
export OPENFAAS_URL=http://127.0.0.1:31112
```

You can use that address directly on the VM during an SSH session, but you'll also be able to access the gateway from other machines by using the VM's IP i.e. `http://192.168.137.2:31112`.

Get instructions to fetch the password via `arkade info openfaas`.

A multi-node setup is possible, but in that case, it's better to set up K3s using K3sup Pro from outside the VMs, and to follow the [HA K3s example](/examples/ha-k3s) followed by the [OpenFaaS for Kubernetes instructions](https://docs.openfaas.com/deployment/).

## OpenFaaS Pro

Create a [slicer secret](/reference/secrets) for your OpenFaaS Pro license key. This let's you securely pass the license to the slicer VM.

Any secrets the VM or user data script need access to should be added to the `.secrets` folder.

```bash
sudo mkdir .secrets
# Ensure only root can read/write to the secrets folder.
sudo chmod 700 .secrets

sudo -E cp ~/.openfaas/LICENSE .secrets/openfaas-license
```

Create a user data file, `openfaas-pro.sh`, to setup an OpenFaaS Pro cluster.

```bash
#!/bin/bash

export HOME=/home/ubuntu
export USER=ubuntu

cd /home/ubuntu/

(
arkade get kubectl kubectx helm faas-cli k3sup stern --path /usr/local/bin
chown $USER /usr/local/bin/*

mkdir -p .kube
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
  --from-file license=/run/slicer/secrets/openfaas-license

helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update && \
  helm upgrade --install openfaas \
  --install openfaas/openfaas \
  --namespace openfaas \
  -f https://raw.githubusercontent.com/openfaas/faas-netes/refs/heads/master/chart/openfaas/values-pro.yaml

chown -R $USER $HOME

echo "export OPENFAAS_URL=http://127.0.0.1:31112" >> $HOME/.bashrc

)
```

Use `slicer new` to generate a configuration file:

```bash
slicer new openfaas-pro \
  --userdata-file openfaas-pro.sh \
  >  openfaas-pro.yaml
```

For SSH access, use the `--github` flag to specify a GitHub username to import SSH keys from or set the `--ssh-key` flag to add publis SSH keys to the VM . Learn more about [SSH in Slicer](/reference/ssh).

Then start up slicer with the generated config:

```bash
sudo -E slicer up -f openfaas-pro.yaml
```

Connect to the slicer VM over SSH:

```bash
ssh ubuntu@192.168.137.2
```

Run the follwing commands to deploy and invoke a function.

Login to OpenFaaS:

```bash
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

Deploy a function:

```bash
faas-cli store deploy figlet
```

Describe, then invoke the function:

```bash
faas-cli describe figlet
faas-cli invoke <<< "SlicerVM.com"
```

## OpenFaaS CE

OpenFaaS CE is licensed for personal, non-commercial use or a single 60 day commercial trial per company.

The above userdata script can be re-used, with a few options removed. Create the `openfaas-ce.sh` userdata script:

```bash
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
```

Create `openfaas-ce.yaml` clicer configuration file:

```bash
slicer new openfaas-ce \
  --userdata-file openfaas-ce.sh \
  >  openfaas-ce.yaml
```
