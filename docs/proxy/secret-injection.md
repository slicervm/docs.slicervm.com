# Inject secrets into outgoing HTTP(s) requests

The main reason to use Slicer Proxy is to audit traffic from VMs, and to block or allow access to certain endpoints. The secondary reason is to inject secrets into requests without the VM ever having access to the secret's value.

An important note is that the VM still has full use of the privileged resource, so it does not apply policy to _what_ the workload can do, but it does help prevent the secret from being leaked, exfiltrated, or captured by a malicious actor.

## Supported credential types

The `--type` flag passed to `slicer proxy secret create` selects how the proxy stores and injects the credential. The following types are supported today:

| Type                    | Source format                                  | Injected as                                                        |
|-------------------------|------------------------------------------------|--------------------------------------------------------------------|
| `bearer` (default)      | Opaque string                                  | `Authorization: Bearer <value>`                                    |
| `basic`                 | `user:pass` literal                            | `Authorization: Basic base64(user:pass)`                           |
| `oauth-claude`          | `~/.claude/.credentials.json`                  | `Authorization: Bearer <access>` to `api.anthropic.com`            |
| `oauth-codex`           | Codex `auth.json` from `codex login`           | `Authorization: Bearer <access>` to `chatgpt.com`                  |
| `oauth-github-copilot`  | opencode `auth.json` with a `gho_*` token      | `Authorization: token <access>` to `api.githubcopilot.com`         |
| `oauth-xai`             | JSON from `slicer proxy oauth xai`             | `Authorization: Bearer <access>` to `api.x.ai`                     |
| `github-app-user`       | JSON from `slicer proxy oauth github-app-user` | `Authorization: Bearer <access>` to `api.github.com`               |
| `github-app`            | JSON / YAML with `app_id` + private key        | `Authorization: Basic base64(x-access-token:<installation-token>)` to `github.com` |

For every OAuth type the proxy holds the refresh material and rotates the access token in the background, so the VM only ever sees egress through the proxy and never the underlying token.

The credential value comes from one of:

* `--value <literal>` — convenient for short tokens but the value appears in shell history.
* `--value-file <path>` — preferred; the file is read once and trailing whitespace and newlines are stripped.

Pass `--force` to overwrite an existing secret with the same name. Allow rules keep referencing the same secret name.

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
PROXY_TOKEN=$(slicer proxy client create sbox-1)

# Register the upstream token on the proxy (kept off the VM)
echo -n 'sk-1234567890' > /tmp/llama.token
slicer proxy secret create llama-1 \
    --host llama.example.com \
    --type bearer \
    --value-file /tmp/llama.token
rm /tmp/llama.token

# Grant the client access to the host and reference the secret
slicer proxy allow sbox-1 \
    --host 'llama.example.com' \
    --secret llama-1
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
slicer proxy revoke sbox-1 --host 'llama.example.com'
```

## Basic Authentication

Basic Authentication is supported for any HTTP upstream that uses `Authorization: Basic ...`, such as a private container registry, the OpenFaaS gateway admin endpoint etc.

Pass the credential as a literal `user:pass` pair via `--value`, or place it in a file and use `--value-file`:

```bash
slicer proxy secret create gateway-admin \
    --host gw.example.com \
    --type basic \
    --value 'admin:s3cr3t'
```

The proxy base64-encodes the pair and emits `Authorization: Basic base64(user:pass)` on every matching request. Prefer `--value-file` to avoid the credential appearing in shell history.

## OAuth credential adoption

Slicer Proxy adopts OAuth credentials rather than intercepting the OAuth 2.0 flow. The reason is that during an interactive 2.0 flow the VM could otherwise obtain a real access token. With adoption the token is minted on **your host** (or by `slicer proxy oauth`), then handed to the proxy. The proxy then:

1. Persists any refresh token into proxy state (sealed by default).
2. Injects the current access token into matching upstream requests.
3. Refreshes the access token in the background when it nears expiry.

The VM only ever sees `HTTPS_PROXY=...` — it never touches the OAuth refresh token or the access token.

### Claude Code

Use the `.credentials.json` file emitted by Claude Code on the host:

```bash
slicer proxy secret create claude-1 \
  --host api.anthropic.com \
  --type oauth-claude \
  --value-file ~/.claude/.credentials.json
```

The proxy adopts `claudeAiOauth`, refreshes as needed, and injects the `Authorization` header that Claude Code expects.

### Codex with ChatGPT login

After `codex login` on the host:

```bash
slicer proxy secret create codex-1 \
  --host chatgpt.com \
  --type oauth-codex \
  --value-file ~/.codex/auth.json
```

The proxy adopts the access token, refresh token, and account id. Refreshes the token as needed.

### GitHub Copilot via opencode

```bash
slicer proxy secret create copilot-1 \
  --host api.githubcopilot.com \
  --type oauth-github-copilot \
  --value-file ~/.local/share/opencode/auth.json
```

The proxy expects the current `gho_*` token shape, refreshes it when opencode has provided a refresh token.

### xAI Grok

Authenticate with xAI using the built-in OAuth helper to produce adoptable JSON:

```bash
slicer proxy oauth xai > ./xai-oauth.json
slicer proxy secret create xai-1 \
  --host api.x.ai \
  --type oauth-xai \
  --value-file ./xai-oauth.json
```

`slicer proxy oauth xai` starts a local callback server on `127.0.0.1:56121` (override with `--port`), opens the xAI authorization URL on stderr, performs the OAuth exchange, and prints adoptable JSON to stdout. If xAI displays a manual code instead of redirecting, paste it on stdin to complete the flow.

The emitted JSON is a top-level OAuth document:

```json
{
  "type": "oauth-xai",
  "access_token": "...",
  "refresh_token": "...",
  "id_token": "...",
  "token_type": "Bearer",
  "token_endpoint": "https://auth.x.ai/oauth2/token",
  "obtained_at": "2026-05-18T12:34:56Z"
}
```

You can also hand-craft this file from another OAuth client, `access_token` and `refresh_token` are required.

### GitHub App user token

This type is used to obtain a user-context access token for a GitHub App, suitable for endpoints such as Gists or other "act as user" API calls. Enable device flow on the GitHub App, then run:

```bash
slicer proxy oauth github-app-user \
  --client-id Iv1.xxxxx > ./github-app-user.json

slicer proxy secret create gh-user \
  --host api.github.com \
  --type github-app-user \
  --value-file ./github-app-user.json
```

`slicer proxy oauth github-app-user` runs the GitHub device flow and writes adoptable JSON to stdout. Use `--github-url https://github.enterprise.example` to target a GitHub Enterprise Server instance.

The emitted JSON looks like:

```json
{
  "type": "github-app-user",
  "client_id": "Iv1.xxxxx",
  "access_token": "ghu_...",
  "refresh_token": "ghr_...",
  "token_type": "bearer",
  "token_endpoint": "https://github.com/login/oauth/access_token",
  "expires_at": "2026-05-22T12:34:56Z",
  "refresh_token_expires_at": "2026-11-21T12:34:56Z",
  "obtained_at": "2026-05-22T08:34:56Z"
}
```

### GitHub App installation token

This type signs a GitHub App JWT on the proxy, mints an **installation access token**, and injects it as `Authorization: Basic base64(x-access-token:<installation-token>)`. Use it to let a VM `git clone` / `git push` against private repositories without ever holding a Personal Access Token or App private key.

The credential is supplied as YAML. With `private_key_file`, only the path is stored in proxy state — the PEM stays on disk on the host. With inline `private_key`, the PEM is stored in proxy state.

```yaml
# gh-app.yaml
app_id: 12345
private_key_file: /etc/slicer/github-app.pem
# Or, embed the PEM directly:
# private_key: |
#   -----BEGIN RSA PRIVATE KEY-----
#   ...
#   -----END RSA PRIVATE KEY-----
api_endpoint: https://api.github.com
```

Register the secret and grant the client access to `github.com`:

```bash
slicer proxy secret create reviewfn \
  --host github.com \
  --type github-app \
  --value-file ./gh-app.yaml

slicer proxy allow ci-1 \
  --host github.com \
  --secret reviewfn
```

For GitHub Enterprise Server, point `api_endpoint` at the enterprise API root (for example `https://github.example.com/api/v3`).

## Requesting other providers

If you need additional OAuth providers, just request them in the Discord channel.
