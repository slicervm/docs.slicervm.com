# Inject secrets into outgoing HTTP(s) requests

The main reason to use Slicer Proxy is to audit traffic from VMs, and to block or allow access to certain endpoints. The secondary reason is to inject secrets into requests without the VM ever having access to the secret's value.

An important note is that the VM still has full use of the privileged resource, so it does not apply policy to _what_ the workload can do, but it does help prevent the secret from being leaked, exfiltrated, or captured by a malicious actor.

## Bearer Token

Imagine you are running a public endpoint for LLM inference using llama.cpp, perhaps it has been exposed on the Internet through an inlets tunnel.

The URL is `https://llama.example.com/v1/chat/completions`, and the real token is `sk-1234567890`.

From a VM with full Internet access, and direct access to the token, you could run:

```bash
curl -X POST https://llama.example.com/v1/chat/completions \
    -H "Authorization: Bearer sk-1234567890" \
    -H "Content-Type: application/json" \
    -d '{"model": "llama3", "messages": [{"role": "user", "content": "Reply with the word: \"token\""}]}'
```

You should see a response that prints "token".

Next, let's wire up Slicer Proxy, and launch a VM with no Internet access, apart from through the proxy.

```bash
PROXY_TOKEN=$(slicer proxy client create llama-1)
slicer proxy allow llama-1 --host 'llama.example.com'
```

Assuming the host group name is `sbox`, and we have no VMs, launching will give us a host called `sbox-1`:

This will launch a VM with no Internet access, apart from through the proxy.

```bash
slicer vm launch --tag role=llama-1

slicer vm exec sbox-1 \
    --env HTTPS_PROXY="https://proxy:$PROXY_TOKEN@192.168.222.1:3129" \
    -- curl -X POST https://llama.example.com/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "llama3", "messages": [{"role": "user", "content": "Reply with the word: \"token\""}]}'
```

We can then revoke the secret from the client, without deleting the secret itself.

```bash
slicer proxy client revoke llama-1
```

## Basic Authentication

Basic Authentication is supported by Slicer Proxy, follow the guide as per the Bearer Token, but use `--type basic`:

```bash
slicer proxy secret create llama-1 --host 'llama.example.com' --type basic --value 'user:pass'
```

## OAuth

Slicer Proxy currently supports OAuth credential adoption, rather than intercepting and overriding the whole flow.

The reason the secret is adopted, and not injected into an OAuth 2.0 flow, is that during a 2.0 flow, the VM could potentially obtain a real token.

Instead, the token is obtained on your host, then adopted into Slicer Proxy, which will refresh the token as required.

For Claude Code:

```bash
slicer proxy secret create claude-1 \
  --host api.anthropic.com \
  --type oauth-claude \
  --value-file ~/.claude/.credentials.json
```

Codex with ChatGPT login

```bash
slicer proxy secret create codex-1 \
  --host chatgpt.com \
  --type oauth-codex \
  --value-file ~/.codex/auth.json
```

GitHub Copilot via opencode auth

```bash
slicer proxy secret create copilot-1 \
  --host api.githubcopilot.com \
  --type oauth-github-copilot \
  --value-file ~/.local/share/opencode/auth.json
```

> This will create a local file or keychain entry with a `gho_` prefixed access token for GitHub's API.

If you need additional OAuth providers, just request them in the Discord channel.
