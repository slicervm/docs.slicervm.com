# Slicer REST API

This page documents the HTTP API exposed by Slicer for managing micro-VMs, images, disks, and operations.

Some endpoints will fail if a trailing slash is given, i.e. `/nodes` is documented, however `/nodes/` may return an error.

## Authentication

No authentication, ideal for local/dev work:

```yaml
config:
  api:
    bind_address: 127.0.0.1
    port: 8080
```

For production:

```yaml
config:
  api:
    bind_address: 127.0.0.1
    port: 8080
    auth:
      enabled: true
```

The token will be saved to: `/var/lib/slicer/auth/token`.

Send an `Authorization: Bearer TOKEN` header for authenticated Slicer daemons.

If you intend to expose the Slicer API over the Internet using something like a [self-hosted inlets tunnel](https://inlets.dev/), or [Inlets Cloud](https://cloud.inlets.dev), then make sure you use the TLS termination option.

i.e.

```bash
DOMAIN=example.com
inletsctl create \
    slicer-api \
    --letsencrypt-domain $DOMAIN \
    --letsencrypt-email webmaster@$DOMAIN
```

Alternatively, if Slicer is on a machine that's public facing, i.e. on Hetzner or another bare-metal cloud, you can use Caddy to terminate TLS. Make sure the API is bound to 127.0.0.1:

Caddyfile:

```
{
    email "webmaster@example.com"
    # Uncomment to try the staging certificate issuer first
    #acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
    # The production issuer:
    acme_ca https://acme-v02.api.letsencrypt.org/directory
 }

slicer-1.example.com {
 reverse_proxy 127.0.0.1:8080
}
```

You can install Caddy via `arkade system install caddy`

## Conventions

- Content-Type: application/json unless noted.
- Timestamps: RFC3339.
- IDs: opaque strings returned by the API.
- Errors: JSON: { "error": "message", "code": "optional_code" } with appropriate HTTP status.

## API Reference

<swagger-ui src="./openapi.yaml"/>
