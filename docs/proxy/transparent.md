# Transparent Proxy Helper

The most explicit way to use Slicer Proxy is to use a `HTTP_PROXY` or `HTTPS_PROXY` environment variable when using programs that need Internet access from within the VM.

For when this becomes cumbersome, you can use the transparent proxy helper.

Note: Slicer on Linux may require the `sudo` command when talking to the API.

## Example with apt-get update and install

```bash
PROXY_TOKEN=$(slicer proxy client create web-1)

slicer proxy allow web-1 --host cloudflare-dns.com  --method POST --path /dns-query
slicer proxy allow web-1 --host archive.ubuntu.com  --method GET  --path '/ubuntu/*'
slicer proxy allow web-1 --host security.ubuntu.com --method GET  --path '/ubuntu/*'

cat > userdata.sh <<EOF
#!/bin/bash
set -eux
export HTTPS_PROXY="http://proxy:${PROXY_TOKEN}@192.168.222.1:3128"

# Install the proxy helper for DNS and outgoing TCP traffic:
/usr/local/bin/slicer-agent proxy install

DEBIAN_FRONTEND=noninteractive apt-get update -y
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
EOF

VM=$(slicer vm launch lab --userdata-file userdata.sh --wait-userdata --json | jq -r '.hostname')

slicer vm exec "$VM" -- systemctl is-active nginx
slicer vm exec "$VM" -- curl -sS http://localhost | head -3
```

## Caveats for Docker

The transparent proxy helper is compatible with Docker, however there are some limitations, which are not specific to Slicer Proxy itself.

Container images use their own root filesystems and userspace, so the CA injection performed by Slicer is not available to them out of the box.

Image pulling should just work, because the helper sets up `/etc/docker/daemon.json` with the proxy configuration.

But for builds, you will need to add the CA certificate at `/runner/ca.crt` into any base images that need to use the Internet.

I.e. to install nginx, you could use the following Dockerfile:

```Dockerfile
FROM ubuntu:22.04

COPY /runner/ca.crt /usr/local/share/ca-certificates/slicer-ca.crt
RUN update-ca-certificates

RUN apt-get update -y
RUN apt-get install -y nginx

CMD ["nginx", "-g", "daemon off;"]
```

Without the step to update the CA certificates, the build will fail with a certificate error.

