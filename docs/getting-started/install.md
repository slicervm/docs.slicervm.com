# Installation

Slicer is evolving, and for the better. Today it uses part of the installation process for [actuated](https://actuated.com).

```bash
mkdir -p ~/.actuated
touch ~/.actuated/LICENSE

(
# Install arkade
curl -sLS https://get.arkade.dev | sudo sh

# Use arkade to extract the agent from its OCI container image
arkade oci install ghcr.io/openfaasltd/actuated-agent:latest --path ./agent
chmod +x ./agent/agent*
sudo mv ./agent/agent* /usr/local/bin/
)

(
cd agent
sudo -E ./install.sh
)
```

Then, get the slicer binary:

```bash
sudo -E arkade octi install ghcr.io/openfaasltd/slicer:latest --path /usr/local/bin
```

Activate Slicer to obtain a license key:

```bash
slicer activate --help

slicer activate
```

