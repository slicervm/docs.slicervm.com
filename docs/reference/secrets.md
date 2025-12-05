# Secrets Management

Slicer provides a secure secrets management system that enables sharing of sensitive data between the host and guest VMs through a VSock-based secrets server. Secrets are stored on the host and synchronized to guest VMs via the slicer-agent service.

## Common Use Cases

The secrets system is ideal for securely sharing database credentials, API keys and tokens, TLS certificates and private keys, SSH keys, and other sensitive configuration data between the host and guest VMs.

## How it works

1) Secrets are created using the CLI or [API](https://docs.slicervm.com/docs/reference/api#manage-secrets) and stored securely on the host filesystem with appropriate permissions.
2) A HTTP secrets server listing on the host VSock is started when launching each VM.
3) The sliser-agent service running in guest VMs connects to the host via VSock to and synchronizes secrets to the guest VM filesystem at `/run/slicer/secrets`.

## Managing Secrets with the CLI

The `slicer secret` command provides a complete interface for managing secrets.

### Create a secret

Create a secret from a literal value:

```bash
slicer secret create my-secret --from-literal="my secret value"
```

Create a secret from a file:

```bash
slicer secret create my-secret --from-file=/path/to/secret.txt
```

Create a secret with custom permissions and ownership:

```bash
slicer secret create my-secret \
  --from-file=/path/to/secret.txt \
  --permissions=0600 \
  --uid=1000 \
  --gid=1000
```

### List secrets

View all secrets:

```bash
slicer secret list
```

### Update a secret

Update a secret from a file:

```bash
slicer secret update my-secret --from-file=/path/to/newsecret.txt
```

Update permissions and ownership:

```bash
slicer secret update my-secret \
  --from-file=/path/to/newsecret.txt \
  --permissions=0644 \
  --uid=1001 \
  --gid=1001
```

### Remove a secret

Remove a secret by name:

```bash
slicer secret remove my-secret
```

## Security Considerations

### File Permissions

By default, secrets are created with `0600` permissions (readable only by the owner). You can specify custom permissions using the `--permissions` flag with octal notation:

- `0600`: Read/write for owner only (default)
- `0644`: Read/write for owner, read-only for group and others
- `0755`: Read/write/execute for owner, read/execute for group and others

### Ownership

Secrets can be assigned to specific users and groups using the `--uid` and `--gid` flags. This is useful when secrets need to be accessible by specific system users or services within the guest VM.

### VSock Communication

The secrets synchronization uses VSock, which provides secure communication between host and guest without network exposure. This ensures secrets are transmitted securely and are not accessible from external networks.

## Guest VM Requirements

For secrets to work properly in guest VMs the slicer-agent services must be running.

The service is automatically included in Slicer VM images and starts during the boot process.
