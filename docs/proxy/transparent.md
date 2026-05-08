# Transparent Proxy Helper

The most explicit way to use Slicer Proxy is to use a `HTTP_PROXY` or `HTTPS_PROXY` environment variable when using programs that need Internet access from within the VM.

For when this becomes cumbersome, you can use the transparent proxy helper.

Note: Slicer on Linux may require the `sudo` command when talking to the API.

The transparent proxy helper uses iptables, so requires the regular non-min images. We will update the min image to be compatible in a future release.

## Example with apt-get update and install

Start with a Slicer config similar to [the Linux guide](/proxy/linux), with the host group of `sbox` and the daemon already running.

We'll do the following:

* Create a client for the proxy, and keep track of its token
* Define restrictive allow rules that mean the client can only install apt packages from the Ubuntu archive.
* Launch a VM and pass it the client's token, and have it install nginx through the proxy helper

Create a client and setup allow rules:
```bash
PROXY_TOKEN=$(slicer proxy client create web-1)

slicer proxy allow web-1 --host cloudflare-dns.com \
    --method POST --path /dns-query
slicer proxy allow web-1 --host archive.ubuntu.com  \
    --method GET --path '/ubuntu/*'
slicer proxy allow web-1 --host security.ubuntu.com \
    --method GET --path '/ubuntu/*'
```

Create a userdata script to install the proxy helper and install nginx through it:

```bash
cat > userdata.sh <<EOF
#!/bin/bash
set -eux

/usr/local/bin/slicer-agent proxy install \
    192.168.222.1 --token ${PROXY_TOKEN}

DEBIAN_FRONTEND=noninteractive apt-get update -y
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
EOF
```

Launch the VM, blocking until the userdata script has finished, then check the status of the nginx service and make a request to it:

```bash
VM=$(slicer vm launch sbox --userdata-file userdata.sh --wait-userdata --json | jq -r '.hostname')

slicer vm exec "$VM" -- systemctl is-active nginx
slicer vm exec "$VM" -- curl -sS http://localhost | head -3
```

You can also use explicit environment variables when setting up the proxy helper:

```bash
cat > userdata.sh <<EOF
#!/bin/bash
set -eux

export HTTPS_PROXY="https://proxy:${PROXY_TOKEN}@192.168.222.1:3129"
export HTTP_PROXY="http://proxy:${PROXY_TOKEN}@192.168.222.1:3128"

/usr/local/bin/slicer-agent proxy install
EOF
```

The `slicer-agent proxy install` command needs root privileges, so if you want to install it outside of userdata, you need to run the command with `sudo`.

## Set up TCP tunnels to traverse the proxy

The transparent helper redirects normal TCP traffic through the proxy, but you can also define a local tunnel. The agent opens a local port inside the VM and forwards it through Slicer Proxy using CONNECT passthrough.

The proxy must have an allow rule for the destination host and port with `--passthrough`.

On the Slicer host:

```bash
sudo slicer proxy allow build-1 \
    --host db.internal.example.com \
    --port 5432 \
    --passthrough \
    --url ./slicer.sock
```

Then inside the VM:

```bash
sudo slicer-agent proxy tunnel add pg \
    --listen 127.0.0.1:5432 \
    db.internal.example.com:5432
```

Then you can connect with i.e. `psql`:

```bash
psql -h 127.0.0.1 -p 5432
```

Tunnels can be managed with:

* `sudo slicer-agent proxy tunnel list` to list tunnels
* `sudo slicer-agent proxy tunnel remove NAME` to remove a tunnel


## Forward SSH through the proxy

You can also forward SSH through the proxy by using slicer-agent proxy connect as a ProxyCommand, either inline or in ~/.ssh/config.

The SSH host needs an allow rule for port 22 with `--passthrough`.

On the Slicer host:

```bash
sudo slicer proxy allow build-1 \
    --host bastion.internal.example.com \
    --port 22 \
    --passthrough \
    --url ./slicer.sock
```

Inside the VM:

```bash
ssh -o ProxyCommand='slicer-agent proxy connect %h:%p' \
    user@bastion.internal.example.com
```

Or in `~/.ssh/config`:

```
  Host bastion
      HostName bastion.internal.example.com
      User user
      ProxyCommand slicer-agent proxy connect %h:%p
```

Then you can connect to the SSH host via `ssh user@bastion.internal.example.com`.

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

