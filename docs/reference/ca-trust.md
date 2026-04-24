# Per-host-group CA trust

Slicer can inject one or more certificate authorities into every VM in
a host group so that anything inside the VM — curl, coding agents,
language runtimes — trusts TLS certs signed by those authorities
without needing `-k`, `--cacert`, or manually edited trust stores.

Use this when:

- you're running a local HTTPS service (a dev server, a credential
  proxy, a private git mirror) and want VMs to reach it over TLS,
- you've adopted an external tool that hands you a CA cert to install
  on clients (Fly tokenizer, Agent Vault, an internal PKI, etc.),
- you want ephemeral sandbox VMs to trust a short-lived dev cert
  without having to re-bake the image.

Nothing here requires a public DNS name or a public cert authority —
it's all local-trust.

## Two knobs

`ca.generate` asks slicer to **mint** a CA for this host group. Useful
when you own both sides of the trust relationship and just want a
ready-to-use root.

`ca.files` asks slicer to **import** one or more pre-existing CA certs
into the trust store. Useful for external tools that ship a CA.

```yaml
host_groups:
  - name: agents
    ca:
      generate: true          # optional: slicer mints a CA for this hg
      files:                  # optional: trust external CAs too
        - ./fly-tokenizer-ca.pem
        - /etc/acme/root.pem
```

You can set either, both, or neither. The VM ends up trusting whatever
you configure, additively.

## What happens at boot

On `slicer up`, for each host group that opts in:

- `generate: true` loads-or-creates `./.slicer/ca/<hg>/ca.{crt,key}`
  in the slicer CWD. Reuses existing material if still valid,
  regenerates if about to expire. 3-year validity by default.
- `files:` reads each path (absolute, or relative to the `slicer up`
  CWD) and stages its bytes alongside the generated CA, if any.

On VM create, the resulting concatenated PEM is written to
`/runner/ca.crt` on the VM's config drive. On first boot
`configure-dns.sh` runs `slicer-agent ca install`, which:

- splits the input into individual `CERTIFICATE` PEM blocks,
- content-hashes each block (first 12 hex chars of its SHA-256) as a
  short ID,
- writes `/etc/ssl/certs/slicer-agent-<short>.pem` (or
  `/etc/pki/tls/certs/…` on RHEL/Rocky) and the OpenSSL hash
  symlinks next to it,
- appends a marker block (`# SLICER_DYNAMIC_CA <short>`) to the
  combined bundle,
- records the install in `/etc/slicer/ca-trust/<short>.installed`
  so the next run is a dedup no-op.

Runs on Debian, Ubuntu, Alpine, Arch and RHEL/Rocky — the agent
detects the trust-store layout at startup and writes to the correct
paths for each.

## Inspect and manage

From the host:

```sh
slicer vm exec <hg>-1 -- slicer-agent ca list
slicer vm exec <hg>-1 -- slicer-agent ca show <short>
slicer vm exec <hg>-1 -- slicer-agent ca remove <short>
slicer vm exec <hg>-1 -- slicer-agent ca wipe
```

Or inside the VM (`slicer vm shell <hg>-1`):

```sh
slicer-agent ca list
slicer-agent ca list --json
```

The `install` subcommand is what `configure-dns.sh` calls at boot and
is safe to re-run — same bytes → no-op, new bytes → new entry.

## Issuing leafs

### Slicer-minted CA

If you opted into `generate: true`, `slicer ca issue` signs server
certs from that CA:

```sh
sudo slicer ca issue \
  --hostgroup agents \
  --host 192.168.137.1 \
  --out server
sudo chown "$USER:$USER" server.crt server.key
```

`--host` accepts an IP (IP SAN) or DNS name (DNS SAN), and is
repeatable for multi-SAN certs. The `sudo` is required because
`slicer up` created `./.slicer/ca/<hg>/` with `0700 root:root`.

### External CA (bring-your-own)

If the CA is yours — from OpenSSL, Step-CA, Vault, Agent Vault, Fly
tokenizer, any PKI — issue the leaf with whatever tooling you already
use. A pure-openssl example:

```sh
# Root CA
openssl req -x509 -newkey rsa:2048 -sha256 -days 365 -nodes \
    -keyout ca.key -out ca.crt \
    -subj "/O=Example Corp/CN=example-ca"

# Leaf with an IP SAN
openssl req -newkey rsa:2048 -nodes \
    -keyout server.key -out server.csr \
    -subj "/CN=192.168.137.1"
cat > san.cnf <<'EOF'
subjectAltName = IP:192.168.137.1
EOF
openssl x509 -req -in server.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out server.crt -days 90 -sha256 -extfile san.cnf
```

Then reference `ca.crt` in the host group YAML:

```yaml
ca:
  files:
    - ./ca.crt
```

## End-to-end: serve HTTPS, curl from a VM

```sh
# host
openssl s_server -cert server.crt -key server.key -accept 8443 -www
```

```sh
# guest
slicer vm shell <hg>-1
curl https://192.168.137.1:8443/
```

Expect a `200` response and no TLS complaints — no `-k`, no
`--cacert`. The agent inside the VM already trusts the signing CA.

## Day-2: rotation, revocation

The trust store is **append-only** by design. Running `slicer-agent
ca install` with a new CA doesn't remove the old one — agents
in-flight don't lose trust mid-operation.

For compromise-driven revocation (a CA's private key has leaked and
you actually want to untrust it), see the break-glass runbook at
[`docs/ca-break-glass.md`][break-glass]. Two commands:

```sh
slicer vm exec <hg>-1 -- slicer-agent ca remove <short>
# or clear the whole slicer-managed set:
slicer vm exec <hg>-1 -- slicer-agent ca wipe
```

Long-running processes (Node / Python / Go) snapshot the trust store
at startup — after a remove, restart the agent or service so the new,
smaller trust set takes effect.

## Runtime-specific notes

| Runtime                      | How it picks up the injected CAs |
|------------------------------|----------------------------------|
| curl, Go stdlib, Python `ssl`| System trust store — no extra config. |
| Node.js (including Claude Code) | Set `NODE_EXTRA_CA_CERTS=/runner/ca.crt`. Node ignores the OS trust store. |
| Python `requests`            | Set `REQUESTS_CA_BUNDLE=/runner/ca.crt` or `SSL_CERT_FILE=/runner/ca.crt`. |
| Anything else                | `SSL_CERT_FILE=/runner/ca.crt` usually works. |

[break-glass]: https://github.com/openfaasltd/slicer/blob/master/docs/ca-break-glass.md
