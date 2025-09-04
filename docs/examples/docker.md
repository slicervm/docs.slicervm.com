# Run a Docker container in Slicer

There are three ways to try out a container or Docker within Slicer:

1. Start your VM as per the [walkthrough](/getting-started/walkthrough.md), then connect via `ssh` and install Docker inside the VM.
2. Use [Userdata](/tasks/userdata.md) to install Docker and pull your image on first boot.
3. Create a custom base image with Docker pre-installed, and your image pre-pulled, with a systemd service to start it on every boot.

On this page we'll cover options 2 and 3.

Option 2 is the easiest to use, and most portable. The boot up time will be delayed whilst your userdata runs. In option 3, that time is spent on the host once, rather than on the first boot of each VM.

We'll show you a generic approach here, but you can adapt it to your needs, or use another tool like [nerdctl](https://github.com/containerd/nerdctl) or [Podman](https://podman.io/) instead of Docker.

The container we're going to use is a [Docker Registry](https://hub.docker.com/_/registry), but likewise you could run your own applications, a database, Ollama, Grafana, or any other container.

## Install Docker and a container via Userdata

You can use [Userdata](/tasks/userdata.md) to install Docker and pull your image on first boot.

The below is only a partial snippet to show you the relevant changes:

```yaml
config:
    host_groups:
    - name: vm
      userdata: |
        #!/bin/bash
        # Install Docker
        curl -fsSL https://get.docker.com | sh

        # Add user to docker group
        usermod -aG docker ubuntu

        docker run -d -p 5000:5000 --restart=always --name registry registry:3
```

## Create a custom base image for a Docker image

Create a one-shot systemd service to run your container on boot-up.

Call this file `docker-registry.service`:

```ini
[Unit]
Description=Run a Docker registry
After=docker.service
Requires=docker.service
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=0
StartLimitBurst=3

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/docker run -d -p 5000:5000 --restart=always --name registry registry:3
TimeoutStartSec=0
Restart=on-failure
RestartSec=10s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=docker-registry

[Install]
WantedBy=multi-user.target
```

```bash
FROM ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest

RUN curl -fsSL https://get.docker.com | sh && \
    usermod -aG docker ubuntu && \
    docker pull registry:3

COPY docker-registry.service /etc/systemd/system/docker-registry.service

RUN systemctl enable docker-registry
```

Then build the image, and push it to a registry.

Next, customise your `config.yaml` to use your new image:

```yaml
config:
   image: "docker.io/alexellis2/slicer-docker-registry:latest"
```

## Try out your container

If you deployed a Docker registry, you can push and pull images to it from your host.

```bash
docker pull alpine
docker tag alpine 192.168.137.1:5000/alpine
docker push 192.168.137.1:5000/alpine
```

You can use this custom registry for your Slicer images too, by adding `insecure_registry: true` to the `config` section of your VM's YAML file.
