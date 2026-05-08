# Setting up Slicer Proxy on macOS

On a macOS host, there are two host groups for Slicer.

* `slicer` - for a single, long-lived VM. This can make use of "slicer proxy" for auditing and secret injections, but will not be able to block outgoing traffic.
* `sbox` - for ephemeral or persistent VM launches via API/CLI - can apply a blanket drop rule, and force all traffic through the proxy.

Just like on Linux, "slicer proxy" is an additional daemon that runs alongside Slicer.

You will be responsible for starting and configuring the proxy and we recommend running it inside the same working folder as your Slicer daemon in the `~/slicer-mac` folder.

First, edit `slicer-mac.yaml`, and make sure the CA and networking options are uncommented in the `sbox` host group section.

The below is a partial snippet, to show the changes you need to make.

```diff
    - name: sbox
      # Optional, for slicer proxy / custom trust workflows.
      # Set to { generate: true } or include files: [...] when needed.
      ca:
-       generate: false
+       generate: true
      count: 0
      vcpu: 2
      ram_gb: 4
      storage_size: 15G
      rosetta: false
      # Allowed values: none (or empty), shutdown, prevent, suspend
      sleep_action: prevent
      share_home: ""
      userdata: ""
      network:
        mode: nat
        gateway: 192.168.64.1/24
+       dns_servers: ["127.0.0.1", "127.0.0.1"]
        # Default is no managed PF filtering. To force sbox traffic through a
        # host proxy, opt in with:
-       # allow:
+       allow:
+         - 192.168.64.1:3128
+         - 192.168.64.1:3129
-       #   - 192.168.64.1:3128
-       #   - 192.168.64.1:3129
-       # drop:
-       #   - 0.0.0.0/0
+       drop:
+         - 0.0.0.0/0
```

* For the networking layer, we enable default deny for all traffic, and only allow the proxy endpoint to be reached on specific ports. If you leave this off, VMs may be able to connect to the SSH port for instance on the host.
* We make sure the DNS servers (which are now blocked), redirect to localhost for a quick rejection. When you start running the [transparent proxy](/proxy/transparent), the helper will redirect DNS queries to slicer proxy on the host.
* We enable CA generation and injection. Without this setting, the certificates minted by the proxy would not be trusted by the guest VMs.

If slicer-mac is already running, you'll need to shut down your VMs gracefully, and restart the daemon with the new config.

## Enable `pf` rules to force all traffic through the proxy

This is a one-time operation, and does require `sudo` privileges.

```bash
sudo ~/slicer-mac/slicer-mac pf apply
```

If you want to revert the `pf` rules later on, to give `sbox` VMs full access to the Internet, run:

```bash
sudo ~/slicer-mac/slicer-mac pf remove
```

## Start slicer and the proxy

You'll need two terminals for this process, or a tmux window split into two panes.

```bash
cd ~/slicer-mac
slicer proxy up \
    --hostgroup sbox \
    --bind 0.0.0.0 \
    --san 192.168.64.1 \
    --seal-key-file ./.slicer/proxy/mk \
    --deny-cidr 192.168.1.0/24
```

This will start the proxy on the host, and allow VMs to access it via the NAT gateway's IP and ports `3128` and `3129`.

One key setting is `--deny-cidr 192.168.1.0/24` - this blocks the proxy from contacting the local LAN range on the host. You may also wish to include other ranges here, including 127.0.0.1.

For discovering the exact HTTPS paths a workload tries to reach before you create allow rules, see [strict and audit modes](/proxy/overview/#strict-and-audit-modes).

Now, start up Slicer in the other terminal:

```bash
cd ~/slicer-mac
slicer-mac up
```

## Test the setup in audit mode only

The default is to deny all egress from the VM, and only allow access to the proxy. The proxy's default is also to deny all requests.

Run the following to allow open Internet access, whilst logging requests:

* Create a client for the proxy, and keep track of its token
* Define an allow rule
* Launch a VM and pass it the client's token, and have it make a HTTP request

```bash
export SLICER_URL="./slicer.sock"
PROXY_TOKEN=$(slicer proxy client create audit-test)
slicer proxy allow audit-test --host '*'
```

Then launch a VM:

```bash
slicer vm launch --tag role=audit-test
```

Execute a command without any proxy credentials, and see it blocked:

```bash
slicer vm exec sbox-1 -- curl -iSL https://wikipedia.org
```

You'll get an error about not being able to resolve DNS.

Then execute a command with the proxy credentials, and see it pass through the proxy:

```bash
slicer vm exec --env HTTPS_PROXY="https://proxy:$PROXY_TOKEN@192.168.64.1:3129" sbox-1 -- curl -iSL https://wikipedia.org
```

You'll see a redirect asking to go to www.wikipedia.org, which means it's working as designed.

## Reset and rotate the CA

Remove the generated CA and key file for the host group:

```bash
rm -rf .slicer/ca/sbox
```

Remove the proxy's leaf certificate and key file:

```bash
rm -rf proxy.cert proxy.key
```

Then re-run the tutorial from the beginning.
