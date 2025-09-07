# Build a custom root filesystem

You can customise a Slicer VM in two ways:

1. Via [userdata on first boot](/tasks/userdata) (a bash script included via the config file)
2. By extending an existing root filesystem with Docker and adding various `COPY` and `RUN` statements

When building within Docker, you must bear in mind that not all of the files shown to you will persist into the final image.

Additionally, if you're copying and pasting commands from installation guides, certain `systemctl` commands will need to be adapted.

If a guide gave you `systemctl enable --now bind` for a custom DNS server, you'd simply remove the `--now` flag. The meaning of `--now` is to instruct the build container to start a systemd service immediately - that's not possible or desirable during a build.

In short, if you are trying to extend a Slicer image, treat it as if it were a Docker container, and only run commands that you know work in a regular Dockerfile.

For anything that does not run or work within a Dockerfile, either use userdata to run the steps on first boot, or add a one-shot systemd unit file that will run your custom steps on first boot.

## Build a custom image

First, refer to the image you want to customise - whether it's for aarch64 or x86_64.

Then create a Dockerfile with a `FROM` line specifying the base image you want to use. For example:

```Dockerfile
FROM ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest
```

If you wanted to install Nginx and have it start automatically, with a website you've created, you could add the following lines to your Dockerfile.

Nginx will run in systemd, we should not try to change the CMD instruction.

```diff
+RUN apt-get update && apt-get install -y nginx
+RUN systemctl enable nginx
+COPY ./my-website /var/www/html
```

Then build and publish the image to your own registry:

```bash
docker build -t docker.io/alexellis2/slicer-nginx:5.10.240-x86_64 .
docker push docker.io/alexellis2/slicer-nginx:5.10.240-x86_64
```

Then edit your Slicer YAML and replace the `image:` with `docker.io/alexellis2/slicer-nginx:5.10.240-x86_64`.

If you wanted Docker to be pre-installed into all your VMs, with the default user already set up for access, you could write:

```diff
+RUN curl -sLS https://get.docker.com | sh
+RUN usermod -aG docker ubuntu
```

If you want to run a local registry, without TLS authentication enabled, you can do so with the following within your YAML file:

```yaml
config:
  insecure_registry: true
```

Then if you want to run a temporary [Docker registry](https://hub.docker.com/_/registry) on another machine on your network:

```bash
docker run -d -p 5000:5000 --restart always \
  --name registry registry:3
```
