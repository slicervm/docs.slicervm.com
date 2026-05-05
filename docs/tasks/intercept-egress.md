# Intercept VM egress

When an agent runs inside a slicer VM — a coding agent, a scripted
task, anything making outbound HTTP(S) calls — you often want a
gatekeeper between it and the public internet. Typical reasons:

- **Block specific domains.** No `wikipedia.org` from sandboxes.
- **Egress restriction.** Only allow an explicit host list, reject
  the rest.
- **Auditing.** Log which URLs the agent touched, with what status.

Squid with SSL-bump is the plainest way to do all three. It's an
industry-standard forward proxy; configured to terminate TLS with a
CA slicer has already generated, it can see the plaintext URL of
every HTTPS request the VM makes and apply ACLs against it. The VM
auto-trusts the on-the-fly leaf certs Squid mints because slicer
injected the signing CA into its trust store at boot — no `-k`, no
per-agent config.

## Prereqs

- **Squid with TLS support** on the host. On Debian/Ubuntu:
  `sudo apt install squid-openssl` (the `squid` package without TLS
  won't do SSL-bump). On Fedora/RHEL: `sudo dnf install squid`.
- Slicer with a host group configured for an isolated network — this
  walkthrough uses the bridge gateway `192.168.137.1` as the proxy IP.

## Slicer config

Generate a host group config with `--ca` so slicer mints a CA, then
patch in the egress lockdown so the proxy is the only reachable host:

```sh
slicer new agents --cidr 192.168.137.0/24 --count 0 --ca > agents.yaml
```

```diff
       network:
-        mode: "bridge"
-        bridge: bragents0
-        tap_prefix: agents
-        gateway: 192.168.137.1/24
+        mode: "isolated"
+        drop: ["0.0.0.0/0"]
+        allow: ["192.168.137.1/32"]
+      dns_servers: ["127.0.0.1"]
```

`dns_servers: ["127.0.0.1"]` makes VM-side DNS fail locally — no
lookups leak out onto the host network. Squid handles all DNS for
proxied traffic.

Start the daemon:

```sh
sudo -E slicer up ./agents.yaml
```

Slicer creates the CA at `./.slicer/ca/agents/ca.{crt,key}`.

## Stage the CA for Squid

Squid needs to read both the cert and the key. Copy them where its
service user can read them:

```sh
sudo install -m 0644 ./.slicer/ca/agents/ca.crt /etc/squid/slicer-ca.crt
sudo install -m 0600 ./.slicer/ca/agents/ca.key /etc/squid/slicer-ca.key
sudo chown proxy:proxy /etc/squid/slicer-ca.*
```

(`proxy:proxy` is Squid's default service account on Debian/Ubuntu.
On Fedora/RHEL it's `squid:squid`.)

## Initialise Squid's SSL DB

Squid keeps the leaves it mints in a small cert DB. Create it once:

```sh
sudo rm -rf /var/spool/squid/ssl_db
sudo /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
sudo chown -R proxy:proxy /var/spool/squid/ssl_db
```

## Squid config

Put this in `/etc/squid/squid.conf` (replace the whole file, or merge
with your existing config):

```squid
http_port 3129 \
    ssl-bump \
    cert=/etc/squid/slicer-ca.crt \
    key=/etc/squid/slicer-ca.key \
    generate-host-certificates=on \
    dynamic_cert_mem_cache_size=4MB

sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB
sslcrtd_children 5

# Bump everything — terminate the client's TLS with a leaf we mint,
# see the plaintext request, then re-encrypt to the upstream.
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# Ban list — matches on the CONNECT target / Host header.
acl banned dstdomain .wikipedia.org

http_access deny banned
http_access allow all

access_log /var/log/squid/access.log squid
```

Reload Squid:

```sh
sudo systemctl restart squid
sudo ss -tln | grep 3129    # confirm Squid is listening
```

## Launch a VM and test

```sh
slicer vm add agents --wait
slicer vm shell agents-1
```

Inside the VM:

```sh
export HTTPS_PROXY=https://192.168.137.1:3129
export NODE_EXTRA_CA_CERTS=/runner/ca.crt   # Node-only; see reference

# Allowed — 200, TLS validates cleanly:
curl -sS -o /dev/null -w '%{http_code}\n' https://example.com/

# Banned — Squid replies 403 with an error page signed by slicer's CA:
curl -sS -o /dev/null -w '%{http_code}\n' https://en.wikipedia.org/
```

Expected:

- `example.com` → `200`, TLS validated by the slicer CA slicer
  already installed into `/etc/ssl/certs/` at first boot.
- `wikipedia.org` → `403` from Squid. No TLS complaints — Squid
  presents a leaf signed by the same trusted CA, the agent sees a
  normal HTTP error response rather than a connection drop.

Check `/var/log/squid/access.log` for the request record, including
the URL path (visible because Squid terminated the TLS).

## Extending

- **Path-level rules.** Squid ACLs can match `url_regex`, so you can
  ban specific URL patterns even within an allowed domain.
- **Allow-list only.** Replace `http_access allow all` with
  `http_access allow <acl>; http_access deny all` and define an
  allow-list of `dstdomain`s.
- **Credential injection.** Squid can rewrite `Authorization` headers
  via [external ACL helpers][squid-ext]. This is the same pattern as
  Fly tokenizer or slicer's own `examples/tokenproxy`, implemented in
  whatever language you prefer.

## See also

- [Trusted CAs in VMs](../reference/ca-trust.md) — schema, BYO-CA
  option, per-runtime env vars, revocation.
- [Squid SSL-bump documentation][squid-bump] — peek, splice, bump
  semantics and the `step` ACL model.

[squid-bump]: https://wiki.squid-cache.org/Features/SslPeekAndSplice
[squid-ext]: https://wiki.squid-cache.org/Features/AddonHelpers
