# Setting up Slicer Proxy on Linux

On a Linux host, "slicer proxy" is an additional daemon that runs alongside Slicer.

You will be responsible for starting and configuring the proxy and we recommend running it inside the same working folder as your Slicer daemon.

For this guide, we'll assume there is one hostgroup with the name of `sbox`.

We only recommend using `slicer proxy` with the `--net=isolated` networking options, with the following setup applied to your Slicer's config.yaml:

```yaml
config:
  host_groups:
  - name: sbox
    count: 0
    ca:
        generate: true
    network:
      mode: isolated
      drop: ["0.0.0.0/0"]
      allow: ["192.168.222.1:3128", "192.168.222.1:3129"]
```

In the above configuration, the "isolated" mode means that VMs cannot talk to each other and are created in their own network namespace. The `drop` rule blocks all outgoing traffic, and then the `allow` rule allows traffic to the proxy on the host.

You can generate a working config file with the following command:

```bash
slicer new sbox --count=0 \
  --net=isolated \
  --drop=0.0.0.0/0 \
  --allow=192.168.222.1:3128 \
  --allow=192.168.222.1:3129 \
  --find-ssh-keys=false \
  --ca \
  --socket ./slicer.sock \
  > slicer.yaml
```

We have bound the API to a UNIX socket, but if you need remote access, you could change this to a TCP port via `--api-bind=127.0.0.1` and `--api-port=8080`, then put a reverse proxy like Caddy in [front of the API](/reference/api) in front of it to terminate TLS.

## Pre-generate a CA for the proxy and slicer

```bash
sudo slicer ca init --hostgroup sbox
```

This will write a private key and public certificate to `./.slicer/ca/sbox/`.

## Bind address and dummy adapter setup

Since there is no bridge or stable IP address for the proxy in "isolated" networking mode, bind the proxy to a host-only address that the isolated VMs are allowed to reach.

On Linux, `slicer proxy up --bind` can set this up for you:

* If `--bind` is a concrete IPv4 address, such as `192.168.222.1`, Slicer checks whether a local adapter already owns it. If not, Slicer creates or updates a per-IP dummy adapter with `192.168.222.1/32`.
* If `--bind` is a CIDR, such as `192.168.222.1/24`, Slicer creates or updates the per-IP dummy adapter with that exact prefix and then listens on `192.168.222.1`.
* If the address already exists on another local adapter, Slicer leaves networking alone and just binds the proxy listener.

The automatic dummy adapter setup requires `CAP_NET_ADMIN`, so run the proxy with `sudo`.

The generated adapter name is stable for the bind IP, so more than one proxy instance can run with different bind addresses.

## Start slicer and the proxy

You'll need two terminals for this process, or a tmux window split into two panes.

```bash
sudo slicer proxy up \
    --hostgroup sbox \
    --bind 192.168.222.1 \
    --deny-cidr 192.168.1.0/24
```

This will start the proxy on the host, and allow VMs to access it via the `192.168.222.1:3128` and `192.168.222.1:3129` ports.

One key setting is `--deny-cidr 192.168.1.0/24` - this blocks the proxy from contacting the local LAN range on the host. You may also wish to include other ranges here, including 127.0.0.1.

Now, start up Slicer in the other terminal:

```bash
sudo slicer up ./slicer.yaml
```

## Audit mode for discovering rules

By default, `slicer proxy up` runs in strict mode. Unknown TLS `CONNECT` requests are denied early, so the logs can only show the target host and port.

When you are porting a workload to Slicer Proxy and need to discover the exact HTTPS paths it requests, start the proxy in audit mode:

```bash
sudo slicer proxy up \
    --hostgroup sbox \
    --bind 192.168.222.1 \
    --deny-cidr 192.168.1.0/24 \
    --mode=audit
```

Audit mode still denies by default. For unknown HTTPS requests, it accepts the connection far enough to terminate TLS with the Slicer Proxy CA, logs the first inner method and path, then returns `403` without forwarding upstream.

For example:

```text
deny client=reviewfn method=GET scheme=https host=api.example.com port=443 path=/v1/models mode=audit reason=no-rule
```

If the client does not trust the Slicer Proxy CA, or pins the upstream certificate, audit mode fails closed: the request is not forwarded and the proxy can only log the host/TLS handshake failure.

## Test the setup in audit mode only

The default is to deny all egress from the VM, and only allow access to the proxy. The proxy's default is also to deny all requests.

Run the following to allow open Internet access, whilst logging requests:

* Create a client for the proxy, and keep track of its token
* Define an allow rule
* Launch a VM and pass it the client's token, and have it make a HTTP request

```bash
export SLICER_URL="./slicer.sock"
PROXY_TOKEN=$(slicer proxy client create audit-test)
sudo slicer proxy allow audit-test --host '*'
```

Then launch a VM:

```bash
# slicer vm launch --tag role=audit-test

VM created
  Hostname: sbox-1
```

Execute a command without any proxy credentials, and see it blocked:

```bash
sudo slicer vm exec sbox-1 -- curl -iSL https://wikipedia.org
```

You'll get an error about not being able to resolve DNS.

Then execute a command with the proxy credentials, and see it pass through the proxy:

```bash
sudo  slicer vm exec \
    --env HTTPS_PROXY="https://proxy:$PROXY_TOKEN@192.168.222.1:3129" \
    sbox-1 -- curl -iSL https://wikipedia.org
```

You'll see a redirect asking to go to www.wikipedia.org, which means it's working as designed.

## Reset and rotate the CA

Remove the generated CA and key file for the host group:

```bash
sudo rm -rf .slicer/ca/sbox
```

Remove the proxy's leaf certificate and key file:

```bash
sudo rm -rf proxy.cert proxy.key
```

Then re-run the tutorial from the beginning.
