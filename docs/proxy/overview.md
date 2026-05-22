# Slicer Proxy

Slicer's proxy lets you inspect and filter HTTP requests egressing your microVMs, and keep secrets away from workloads.

## Features

* Log and audit outgoing HTTP(s) requests
* Inject credentials into outgoing requests - Bearer Token, Basic Auth, and adopted OAuth for Claude Code, Codex (ChatGPT), GitHub Copilot, xAI Grok, GitHub App user tokens, and GitHub App installation tokens
* Block specific domains, URLs, paths, and methods
* Rule expiry - set a TTL for rules to automatically expire after a certain time
* Passthrough mode for unmodified requests, or TCP access such as Postgresql or SSH
* Audit mode for discovering the HTTPS paths a workload tries to reach before you create allow rules
* Transparent proxying helper, for when proxy variables cannot be set or programs do not honour them 

Rules can be driven through the CLI, REST API, or with code using the SDKs.

## Concepts

Slicer separates the nouns. Register a client, register secrets, then bind them for a period and let them expire. Clients are identified by the auth header they send to the proxy — no fake secrets, no source-IP heuristics.

```
┌─ client ──────────┐         ┌─ secret ─────────────┐
│  name             │         │  name                │
│  token            │         │  type    bearer      │
└─────────┬─────────┘         │          basic-auth  │
          │                   │          oauth       │
          │ owns              │  value               │
          │                   └──────────┬───────────┘
          │                              │
          │                              │ optional
          │                              │
┌─────────▼──────────────────────────────▼───────────┐
│  allow rule                                        │
│                                                    │
│  host                                              │
│  paths                                             │
│  methods                                           │
│  ports                                             │
│  ttl                                               │
└────────────────────────────────────────────────────┘
```
* A client owns zero or more allow rules - no rules means default deny
* An allow rule may be bare for access to a domain for a given TTL (automatic expiry)
* Or an allow rule may also reference a secret — when it does, the proxy injects the credential into matching requests.


## Conceptual workflow

In the diagram below, we see a microVM that has been booted up, and given a `HTTPS_PROXY` value with a specific authentication token for the proxy. This is how we identify the microVM and client.

```
┌─ Host ──────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─ Slicer + API ─────┐    ┌─ Slicer Proxy ─────────────────────┐  │
│   │  Control plane     │    │  Dummy adapter  192.168.222.1      │  │
│   │                    │    │                                    │  │
│   │                    │    │   :3128   HTTP_PROXY               │  │
│   │                    │    │   :3129   HTTPS_PROXY              │  │
│   └────────────────────┘    └────────────────────────────────────┘  │
│                                            ▲                        │
└────────────────────────────────────────────┼────────────────────────┘
                                             │ egress
┌─ Network namespace · isolated mode ────────┼────────────────────────┐
│                                            │                        │
│   ┌─ microVM ──────────────────────────────┴─────────────────────┐  │
│   │                                                              │  │
│   │   Drop   0.0.0.0/0                                           │  │
│   │   Allow  192.168.222.1:3128, 192.168.222.1:3129              │  │
│   │                                                              │  │
│   │   HTTPS_PROXY=https://proxy:spi_REDACTED@192.168.222.1:3129  │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

This gives us the following workflow when we want to launch a microVM with filtered egress policy:

1. Define a client and keep track of its token
2. Define any secrets it may need i.e. an LLM token
3. Grant the client access to specific websites, and optionally reference the secret
4. Then boot up the VM, and pass in the proxy reference such as `HTTPS_PROXY=https://proxy:spi_REDACTED@192.168.222.1:3129`

For more dynamic workloads, the grants can be made during execution and are effective immediately.

## Strict and audit modes

Slicer Proxy has two operating modes:

* `--mode=strict` (default) denies unknown HTTPS `CONNECT` requests immediately, and logs only the destination host and port.
* `--mode=audit` still denies by default, but accepts unknown HTTPS requests far enough to terminate TLS, log the first inner method and path, then return `403` without forwarding upstream.

When you are porting a workload and need to discover the exact HTTPS paths it requests, start the proxy in audit mode:

=== "Linux"

    ```bash
    sudo slicer proxy up \
        --hostgroup sbox \
        --bind 192.168.222.1 \
        --deny-cidr 192.168.1.0/24 \
        --mode=audit
    ```

=== "macOS"

    ```bash
    cd ~/slicer-mac

    slicer proxy up \
        --hostgroup sbox \
        --bind 0.0.0.0 \
        --san 192.168.64.1 \
        --seal-key-file ./.slicer/proxy/mk \
        --deny-cidr 192.168.1.0/24 \
        --mode=audit
    ```

Audit mode uses the Slicer Proxy CA to inspect the first denied HTTPS request.

For example:

```text
deny client=reviewfn method=GET scheme=https host=api.example.com port=443 path=/v1/models mode=audit reason=no-rule
```

Use audit mode to derive allow rules, then restart the proxy without `--mode=audit` for normal strict enforcement.

If the client does not trust the Slicer Proxy CA, or pins the upstream certificate, audit mode fails closed: the request is not forwarded and the proxy can only log the host/TLS handshake failure.

## Passthrough mode

Slicer Proxy can also be used to passthrough TCP traffic without TLS termination.

This is required for:

* Programs which pin a TLS certificate or to a specific CA at build time
* Traffic which breaks for some reason when proxied through "slicer proxy"
* Traffic which is not HTTP(s) i.e. SSH and Postgresql

To create a passthrough rule, run the following command, assuming `client1` already exists:

```bash
slicer proxy allow client1 \
 --host ssh-host.example.com \
 --port 22 \
 --passthrough
```

Then configure the host in `.ssh/config` to use the proxy. Pick the tab that matches the host platform you're running Slicer on:

=== "Linux"

    ```ssh
    Host ssh-host.example.com
        ProxyCommand nc -x 192.168.222.1:3129 %h %p
    ```

=== "macOS"

    ```ssh
    Host ssh-host.example.com
        ProxyCommand nc -x 192.168.64.1:3129 %h %p
    ```

Then you can connect to the host using SSH:

```bash
ssh ssh-host.example.com
```
