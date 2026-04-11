# Build a Custom Image

The default Slicer images ship with a minimal Ubuntu or Rocky Linux installation. If your sandboxes need specific packages - compilers, runtimes, libraries - installing them via userdata on every boot wastes time. A derived image bakes those dependencies in, so VMs start ready to work.

This page covers building a derived image locally. For publishing via CI, see [Publish your own images](/platform/publish-images/). For the full reference on image building, see [Build a custom root filesystem](/tasks/custom-image/).

## How it works

Slicer images are OCI images built with a Dockerfile. You write a `FROM` line pointing at a base Slicer image, add your `RUN` and `COPY` steps, then push the result to a container registry. Slicer pulls the image at startup and uses it for every VM in the host group.

The base images are listed in the [images reference](/reference/images/). For most x86_64 workloads:

```Dockerfile
FROM ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest
```

For arm64:

```Dockerfile
FROM ghcr.io/openfaasltd/slicer-systemd-arm64:6.1.90-aarch64-latest
```

## Example: Python sandbox

A sandbox image with Python 3, pip, and common data libraries pre-installed:

```Dockerfile
FROM ghcr.io/openfaasltd/slicer-systemd:6.1.90-x86_64-latest

RUN apt-get update -qy \
  && apt-get install -qy \
    python3 \
    python3-pip \
    python3-venv \
  && apt-get clean
```

Build and push:

```bash
docker build -t ghcr.io/your-org/slicer-python:6.1.90-x86_64 .
docker push ghcr.io/your-org/slicer-python:6.1.90-x86_64
```

Then reference it in your Slicer YAML:

```yaml
config:
  host_groups:
    - name: sandbox
      count: 0
      # ...
  image: "ghcr.io/your-org/slicer-python:6.1.90-x86_64"
```

## Things to know about Dockerfile builds

Slicer images run systemd as PID 1. During the Docker build, systemd is not running, so:

* Use `systemctl enable SERVICE` to configure a service to start on boot. Do not use `--now`
* Avoid commands that need a running kernel or network stack
* Do not change the `CMD` or `ENTRYPOINT`. Slicer manages the boot process

For anything that cannot run inside a Dockerfile (e.g. commands that need networking or a running init system), use [userdata](/tasks/userdata/) to run the steps on first boot, or add a one-shot systemd unit.

See [Build a custom root filesystem](/tasks/custom-image/) for more detail.

## Watch the architecture

If you are building on a Mac with Apple Silicon, Docker defaults to arm64 images. A Slicer host running on x86_64 cannot boot an arm64 root filesystem, and the failure mode is not obvious. The VM may hang or kernel panic on boot.

Options:

* Use `docker buildx build --platform linux/amd64` to cross-compile locally. This works but is slower due to QEMU emulation inside Docker Desktop.
* Build on a matching architecture: an x86_64 Linux machine, a CI runner, or a Slicer VM itself.

If all your Slicer hosts are arm64 (e.g. Raspberry Pi, Ampere), the reverse applies - build for `linux/arm64`.

Slicer images are not multi-arch. Each architecture needs its own image built natively on matching hardware. If you need to automate this, see [Publish your own images](/platform/publish-images/).

!!! note "Private registries"
    GHCR packages default to private. Either make the package public in GitHub's package settings, or configure `insecure_registry: true` in your Slicer YAML if using a private registry without TLS.

## Test the image

After pushing, restart Slicer with the new image reference and create a sandbox:

```bash
slicer new sandbox \
  --count=0 \
  > sandbox.yaml
```

Edit `sandbox.yaml` and set the `image:` field to your custom image, then start Slicer:

```bash
sudo slicer up ./sandbox.yaml
```

Create a VM and verify the packages are present:

```bash
export TOKEN=$(sudo cat /var/lib/slicer/auth/token)

curl -sf -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://127.0.0.1:8080/hostgroup/sandbox/nodes -d '{}'

# Wait for agent, then check python is installed
curl -sf -H "Authorization: Bearer $TOKEN" \
  -X POST \
  "http://127.0.0.1:8080/vm/sandbox-1/exec?cmd=python3&args=--version&stdout=true"
```

## Wrapping up

Derived images let you trade a one-time build step for faster, more predictable sandbox boot times. Bake in everything your workload needs, publish it to GHCR, and let Slicer pull it on startup.

See also:

* [Build a custom root filesystem](/tasks/custom-image/) - full reference for Dockerfile-based image builds
* [Images for microVMs](/reference/images/) - base image tags and availability
* [Userdata](/tasks/userdata/) - run scripts on first boot for anything that cannot be baked into the image
* [Storage backends](/storage/overview/) - ZFS and devmapper for near-instant boot from snapshots
