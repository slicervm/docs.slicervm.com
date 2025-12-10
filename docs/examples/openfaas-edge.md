# OpenFaaS Edge in slicer

Follow this guide to setup a pre-configured microVM with [OpenFaas Edge](https://docs.openfaas.com/edge/overview/) deployed along with the [OpenFaaS Function Builder API](https://docs.openfaas.com/edge/builder/).

After following the steps in this example you will be able to build and deploy functions to faasd directly from the VM or access the gateway and function builder from other machines by using the VM's IP.

## Userdata

Create `openfaas-edge.sh`.

By default the script installs OpenFaaS Edge a private registry and the [OpenFaaS Function Builder API](https://docs.openfaas.com/edge/builder/) as additional services. Select which services to install by changing the environment variables in the script configuration section.

Note that the function builder addon requires a separate license.

Available configuration options:

| Env | Description | Default|
| --- | ----------- | ------ |
| INSTALL_REGISTRY | Install a private registry | `true` |
| INSTALL_BUILDER | Install the OpenFaaS Function Builder API | `true` |

```bash
#!/usr/bin/env bash

#==============================================================================
# OpenFaaS Edge Installation Script
#==============================================================================
# This script installs OpenFaaS Edge, including optional
# components like a private registry and function builder.
#==============================================================================

set -euxo pipefail

#==============================================================================
# CONFIGURATION
#==============================================================================

# Install additional services (registry and function builder)
export INSTALL_REGISTRY=true
export INSTALL_BUILDER=true

#==============================================================================
# SYSTEM PREPARATION and OpenFaaS Edge Installation
#==============================================================================

has_dnf() {
  [ -n "$(command -v dnf)" ]
}

has_apt_get() {
  [ -n "$(command -v apt-get)" ]
}

echo "==> Configuring system packages and dependencies..."

if $(has_apt_get); then
  export HOME=/home/ubuntu

  sudo apt update -y

  # Configure iptables-persistent to avoid interactive prompts
  echo iptables-persistent iptables-persistent/autosave_v4 boolean false | sudo debconf-set-selections
  echo iptables-persistent iptables-persistent/autosave_v6 boolean false | sudo debconf-set-selections

  arkade oci install --path . ghcr.io/openfaasltd/faasd-pro-debian:latest
  sudo apt install ./openfaas-edge-*-amd64.deb --fix-broken -y

  if [ "${INSTALL_REGISTRY}" = "true" ]; then
    sudo apt install apache2-utils -y
  fi
elif $(has_dnf); then
  export HOME=/home/rocky

  arkade oci install --path . ghcr.io/openfaasltd/faasd-pro-rpm:latest
  sudo dnf install openfaas-edge-*.rpm -y


  if [ "${INSTALL_REGISTRY}" = "true" ]; then
    sudo dnf install httpd-tools -y
  fi
else
    fatal "Could not find apt-get or dnf. Cannot install dependencies on this OS."
    exit 1
fi

# Install faas-cli
arkade get faas-cli --progress=false --path=/usr/local/bin/

# Create the secrets directory and touch the license file
sudo mkdir -p /var/lib/faasd/secrets
touch /var/lib/faasd/secrets/openfaas_license

#==============================================================================
# PRIVATE REGISTRY AND FUNCTION BUILDER SETUP
#==============================================================================

# Always install registry if builder is installed
if [ "${INSTALL_BUILDER}" = "true" ]; then
 INSTALL_REGISTRY=true
fi

if [ "${INSTALL_REGISTRY}" = "true" ]; then
    echo "==> Setting up private container registry..."

    # Generate registry authentication
    export PASSWORD=$(openssl rand -base64 16)
    echo $PASSWORD > $HOME/registry-password.txt

    # Create htpasswd file for registry authentication
    htpasswd -Bbc $HOME/htpasswd faasd $PASSWORD
    sudo mkdir -p /var/lib/faasd/registry/auth
    sudo mv $HOME/htpasswd /var/lib/faasd/registry/auth/htpasswd

    # Create registry configuration
    sudo tee /var/lib/faasd/registry/config.yml > /dev/null <<EOF
version: 0.1
log:
  accesslog:
    disabled: true
  level: warn
  formatter: text

storage:
  filesystem:
    rootdirectory: /var/lib/registry

auth:
  htpasswd:
    realm: basic-realm
    path: /etc/registry/htpasswd

http:
  addr: 0.0.0.0:5000
  relativeurls: false
  draintimeout: 60s
EOF

    # Configure registry authentication for faas-cli
    cat $HOME/registry-password.txt | faas-cli registry-login \
      --server http://registry:5000 \
      --username faasd \
      --password-stdin

    # Setup Docker credentials for faasd-provider
    sudo mkdir -p /var/lib/faasd/.docker
    sudo cp ./credentials/config.json /var/lib/faasd/.docker/config.json

    # Ensure pro-builder can access Docker credentials
    sudo mkdir -p /var/lib/faasd/secrets
    sudo cp ./credentials/config.json /var/lib/faasd/secrets/docker-config

    # Configure local registry hostname resolution
    echo "127.0.0.1 registry" | sudo tee -a /etc/hosts

    echo "==> Adding registry services to docker-compose..."

    # Append additional services to docker-compose.yaml
    sudo tee -a /var/lib/faasd/docker-compose.yaml > /dev/null <<EOF

  registry:
    image: docker.io/library/registry:3
    volumes:
    - type: bind
      source: ./registry/data
      target: /var/lib/registry
    - type: bind
      source: ./registry/auth
      target: /etc/registry/
      read_only: true
    - type: bind
      source: ./registry/config.yml
      target: /etc/docker/registry/config.yml
      read_only: true
    deploy:
      replicas: 1
    ports:
      - "5000:5000"
EOF

fi

if [ "${INSTALL_BUILDER}" = "true" ]; then
    echo "==> Configuring function builder..."

    # Generate payload secret for function builder
    openssl rand -base64 32 | sudo tee /var/lib/faasd/secrets/payload-secret

    echo "==> Adding function builder services to docker-compose..."

    # Append additional services to docker-compose.yaml
    sudo tee -a /var/lib/faasd/docker-compose.yaml > /dev/null <<EOF

  pro-builder:
    depends_on: [buildkit]
    user: "app"
    group_add: ["1000"]
    restart: always
    image: ghcr.io/openfaasltd/pro-builder:0.5.3
    environment:
      buildkit-workspace: /tmp/
      enable_lchown: false
      insecure: true
      buildkit_url: unix:///home/app/.local/run/buildkit/buildkitd.sock
      disable_hmac: false
      # max_inflight: 10 # Uncomment to limit concurrent builds
    command:
     - "./pro-builder"
     - "-license-file=/run/secrets/openfaas-license"
    volumes:
      - type: bind
        source: ./secrets/payload-secret
        target: /var/openfaas/secrets/payload-secret
      - type: bind
        source: ./secrets/openfaas_license
        target: /run/secrets/openfaas-license
      - type: bind
        source: ./secrets/docker-config
        target: /home/app/.docker/config.json
      - type: bind
        source: ./buildkit-rootless-run
        target: /home/app/.local/run
      - type: bind
        source: ./buildkit-sock
        target: /home/app/.local/run/buildkit
    deploy:
      replicas: 1
    ports:
     - "8088:8080"

  buildkit:
    restart: always
    image: docker.io/moby/buildkit:v0.23.2-rootless
    group_add: ["2000"]
    user: "1000:1000"
    cap_add:
      - CAP_SETUID
      - CAP_SETGID
    command:
    - rootlesskit
    - buildkitd
    - "--addr"
    - unix:///home/user/.local/share/bksock/buildkitd.sock
    - --oci-worker-no-process-sandbox
    security_opt:
    - no-new-privileges=false
    - seccomp=unconfined        # Required for mount(2) syscall
    volumes:
      # Runtime directory for rootlesskit/buildkit socket
      - ./buildkit-rootless-run:/home/user/.local/run
      - /sys/fs/cgroup:/sys/fs/cgroup
      # Persistent state and cache directories
      - ./buildkit-rootless-state:/home/user/.local/share/buildkit
      - ./buildkit-sock:/home/user/.local/share/bksock
    environment:
      XDG_RUNTIME_DIR: /home/user/.local/run
      TZ: "UTC"
      BUILDKIT_DEBUG: "1"         # Enable for debugging
      BUILDKIT_EXPERIMENTAL: "1"  # Enable experimental features
    deploy:
      replicas: 1
EOF

fi

#==============================================================================
# INSTALLATION EXECUTION
#==============================================================================

echo "==> Installing faasd..."

# Execute the installation
sudo /usr/local/bin/faasd install

#==============================================================================
# POST-INSTALLATION CONFIGURATION
#==============================================================================

if [ "${INSTALL_BUILDER}" = "true" ]; then
    echo "==> Configuring insecure registry access..."

    # Configure faasd-provider to use insecure registry
    sudo sed -i '/^ExecStart=/ s|$| --insecure-registry http://registry:5000|' \
        /lib/systemd/system/faasd-provider.service

    # Reload systemd and restart faasd-provider
    sudo systemctl daemon-reload
    sudo systemctl restart faasd-provider
fi

echo "==> OpenFaaS Edge installation completed successfully!"
echo ""
echo "1. Access the OpenFaaS gateway at http://localhost:8080"
echo "2. Get your admin password: sudo cat /var/lib/faasd/secrets/basic-auth-password"
if [ "${INSTALL_REGISTRY}" = "true" ]; then
    echo "3. Private registry available at http://localhost:5000"
    echo "4. Registry password: cat $HOME/registry-password.txt"
fi
if [ "${INSTALL_BUILDER}" = "true" ]; then
    echo "5. Pro-builder service available at http://localhost:8088"
fi
```

## VM config

Use `slicer new` to generate a configuration file:

```bash
slicer new openfaas-edge \
  --userdata-file ./openfaas-edge.sh \
  > openfaas-edge.yaml
```
The slicer config will be adapted from the [walkthrough](/getting-started/walkthrough). When you create the YAML, name it `openfaas-edge.yaml`.


Optionally use  the `--ssh-key` or `--github` flags to add an additional ssh key so you can connect via SSH to deploy functions, review the logs, and access check the status of the faasd services.

The userdate script provided in this guide can be used on both ubuntu and rocky. You can change user the `--image` flag to swith the image to your preferred OS. An overview of the available images can be found [here](/reference/images/).

## Run OpenFaaS Edge

Start the VM:

```sh
sudo -E slicer up ./openfaas-edge.yaml`
```

Login to the VM and activate faasd using a static license key are by running faasd activate.

Commercial users can create their license key as follows:

```sh
sudo nano /var/lib/faasd/secrets/openfaas_license"
```

For personal, non-commercial use only, GitHub Sponsors of @openfaas can run:

```sh
sudo faasd github login
sudo faasd activate
```

Restart faasd and the faad-provider after adding the license key:

```sh
sudo systemctl restart faasd faasd-provider
```

## Build and deploy a function within the VM

Login in to the VM over ssh to perform a test build on the VM directly.

Once you connected to the VM authenticate the faas-cli for gateway access:

```sh
sudo cat "/var/lib/faasd/secrets/basic-auth-password" \
 | faas-cli login --password-stdin
```

Scaffold a new function for testing:

```sh
faas-cli new --lang python3-http \
  --prefix registry:5000/functions \
  pytest
```

The `--prefix` flag is used to set prefix for the function image to our local registry.

Build the function using the function builder API and deploy it:

```sh
# Get the payload secret
sudo cp /var/lib/faasd/secrets/payload-secret ./payload-secret

faas-cli up \
    --remote-builder http://127.0.0.1:8088 \
    --payload-secret ./payload-secret \
    --tag=digest
```

## Access OpenFaaS Edge from your own workstation

Get the basic auth password from the VM and login with the faas-cli:

```sh
export OPENFAAS_URL=http://192.168.137.2:8080
ssh ubuntu@192.168.137.2 "sudo cat /var/lib/faasd/secrets/basic-auth-password" \
 | faas-cli login --password-stdin
```

Note that the `OPENFAAS_URL` is configured to use the VM's IP address.

Deploy a function form the function store:

```sh
faas-cli store deploy nodeinfo
```

Invoke the nodeinfo function:

```sh
curl -s http://192.168.137.2:8080/function/nodeinfo
```

If you have the function builder installed you can build and deploy a function using the function builder API:

Scaffold a new function for testing:

```sh
faas-cli new --lang python3-http \
  --prefix registry:5000/functions \
  pytest
```

The `--prefix` flag is used to set the prefix for the function image to our local registry deployed on the VM.


```sh
# Get the payload secret from the VM
ssh ubuntu@192.168.137.2 "sudo cat /var/lib/faasd/secrets/payload-secret" > ./payload-secret

faas-cli up \
  --remote-builder http://192.168.137.2:8088 \
  --payload-secret ./payload-secret \
  --tag=digest
```

The `--remote-builder` flag points to the Function Builder API exposed on the VM.

Test the new function:

```sh
curl -s http://192.168.137.2:8080/function/pytest
```

### Access the registry

The be able to push and pull function from the image registry directly you need to add the VM IP address to your hosts file and login with docker.

Add a new entry to your hosts file that points to the registry:

```sh
echo "192.168.137.2 registry" | sudo tee -a /etc/hosts
```

The registry has to be added as an insecure registry in the docker daemon configuration file, `/etc/docker/daemon.json`:

```sh
{
  "insecure-registries": [ "registry:5000" ]
}
```

Get get the registry password from the VM and login with docker:

```sh
ssh ubuntu@192.168.137.2 "sudo cat ./registry-password.txt" > ./registry-password.txt
cat registry-password.txt | docker login \
  --username faasd \
  --password-stdin \
  http://registry:5000
```

You should now be able to push and pull images from the registry.

You can try this by building a function on your workstation and pushing it to the registry. M

```sh
faas-cli new --lang node22 \
  --prefix registry:5000/functions \
  --append stack.yaml \
  nodetest

faas-cli up
```

Test the new function:

```sh
curl -s http://192.168.137.2:8080/function/nodetest -d "Hello World"
```
