# Trusted CAs in VMs

Make every VM in a host group trust one or more CAs for internal services, or for intercepting proxies such as [slicer proxy](/proxy/overview) or [Squid](https://slicervm.com/blog/intercepting-filtering-agent-traffic/).

## Schema

```yaml
host_groups:
  - name: agents
    ca:
      generate: true           # optional
      files:                   # optional
        - ./my-ca.pem
        - /etc/squid/ssl_cert/squid-ca.pem
```

| Field | Type | Meaning |
|-------|------|---------|
| `ca.generate` | bool | If `true`, slicer loads-or-creates `./.slicer/ca/<hg>/ca.{crt,key}` in its CWD. 3-year validity, regenerated within 7 days of expiry. |
| `ca.files` | `[]string` | List of PEM paths. Absolute, or relative to the `slicer up` CWD. Contents are injected into the guest trust store. |

Either, both, or neither can be set. When both are set, the VM trusts everything — the generated CA and every file — additively. A file may contain multiple `CERTIFICATE` PEM blocks; each is installed separately.

## Built-in CA

When running `slicer new`, the `--ca` flag will generate a local CA authority.

```sh
slicer new demo --cidr 192.168.137.0/24 --count 0 --ca > demo.yaml
sudo -E slicer up ./demo.yaml
sudo slicer ca issue --hostgroup demo --host 192.168.137.1 --out server
sudo chown "$USER:$USER" server.crt server.key
```

The CA lives at `./.slicer/ca/demo/ca.{crt,key}`. `sudo` is required on `slicer ca issue` because that directory is created `0700 root:root`.

## Bring Your Own Certificate Authority

Generate the config without `--ca`, then patch in `ca.files:` pointing at your pre-existing CA. Most often this is outside the slicer CWD — absolute paths are fine:

```diff
       network:
         mode: "bridge"
         bridge: brdemo0
         tap_prefix: demo
         gateway: 192.168.137.1/24
+      ca:
+        files:
+          - /etc/squid/ssl_cert/squid-ca.pem
```

Leaf issuance happens in whichever tool owns the CA (e.g. `step ca certificate …`, `cfssl gencert …`, Squid's SSL-bump, Agent Vault's minter, etc.). Slicer isn't involved in that step.

Set `generate: true` and `files:` together to trust both the Slicer-minted CA and one or more external CAs.

## Mechanics

Slicer reads the CAs at `slicer up`; each VM installs them into its system trust store on first boot. Re-runs are safe and cheap — the install is keyed by content hash, so the same CA never ends up installed twice. Rotation is **append-only**: adding a new CA never removes an older one, so in-flight TLS sessions don't break mid-change. Compromise-driven revocation is an explicit operator action; see below.

## Inspect the CA in a VM

These commands are run inside the VM using Slicer's guest agent:

```sh
slicer vm exec <vm> -- slicer-agent ca list
slicer vm exec <vm> -- slicer-agent ca list --json
slicer vm exec <vm> -- slicer-agent ca show <short>
```

`list` prints one row per installed CA with a 12-char `short` identifier, subject, and `not_after`.

## Per-runtime trust-store behaviour

Most consumers read the system trust store automatically. A few
common exceptions:

| Runtime                      | How it picks up the injected CA |
|------------------------------|---------------------------------|
| curl, Go stdlib, Python `ssl`| System trust store — nothing to set. |
| Node.js (including Claude Code) | `NODE_EXTRA_CA_CERTS=/runner/ca.crt`. Node ignores the OS store. |
| Python `requests`            | `REQUESTS_CA_BUNDLE=/runner/ca.crt` (or `SSL_CERT_FILE`). |
| Anything else                | `SSL_CERT_FILE=/runner/ca.crt` usually works. |

Long-running processes snapshot the trust store at startup; after an install or remove, restart the process to see the new set.

---

## Appendix: revoking a CA

CAs are **append-only** by default — new ones add alongside old ones. A compromised CA stays trusted until you tell each VM to forget it.

List, then revoke by short ID:

```sh
slicer vm exec <vm> -- slicer-agent ca list
# SHORT         SUBJECT                                       NOT_AFTER
# 6de3d548a018  CN=extern-demo-ca, O=Example Corp             2027-04-24
# aeae8c8ee3bc  CN=demo, OU=Slicer CA, O=OpenFaaS Ltd         2029-04-23

slicer vm exec <vm> -- slicer-agent ca remove 6de3d548a018
```

Or wipe everything slicer installed (the OS root bundle stays untouched):

```sh
slicer vm exec <vm> -- slicer-agent ca wipe
```

After revoking, restart any long-running TLS consumer so it re-reads the trust store.

### What revocation doesn't do

- **Doesn't re-issue leafs.** Services presenting a cert chained to the revoked CA will now fail validation — issue a replacement from a different CA (`slicer ca issue` or your external PKI).
- **Doesn't propagate across VMs.** Run against each affected VM, or delete + relaunch if the VMs are ephemeral.
- **Doesn't update the config drive.** On a persistent-disk VM reboot, `/runner/ca.crt` is whatever slicer wrote at create time. If the revoked CA is still in your `ca:` config, it'll be  reinstalled on boot. Remove the file from `ca.files:` (or regenerate `generate: true`'s root with `slicer ca init --force`) before revoking, so the compromised CA doesn't come back in.

