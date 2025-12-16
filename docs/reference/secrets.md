# Secrets Management

Slicer provides a secure secrets management system that enables sharing of sensitive data between the host and guest VMs through a VSOCK-based secrets server. Secrets are stored on the host and synchronized to guest VMs via the `slicer-agent` service.

## Common Use Cases

The secrets system is ideal for securely sharing database credentials, API keys and tokens, TLS certificates and private keys, SSH keys, and other sensitive configuration data between the host and guest VMs.

## How it works

1. Secrets are created manually or through the Slicer CLI or [API](https://docs.slicervm.com/docs/reference/api#manage-secrets). Secrets are stored securely on the host filesystem in the `.secrets` folder.

2. A HTTP secrets server listing on the host VSOCK is started when launching each VM.

3. The `slicer-agent` service running in the guest VM connects to the host via VSOCK and synchronises secrets to the guest VM filesystem at `/run/slicer/secrets`.

## Create secrets

Create the `.secrets` directory if it does not exist and give it the correct permissions.

```bash
sudo mkdir .secrets
# Ensure only root can read/write to the secrets folder.
sudo chmod 700 .secrets
```

Create a new secret file:

```bash
echo "my-api-key" | sudo tee .secrets/api-key > /dev/null
```

Permission and ownership are mapped 1:1 to the guest VM filesystem. Change permissions and ownership accordingly to manage who has access to secrets within the VM:

```bash
sudo chmod 600 .secrets/api-key

# Optionally change ownership
sudo chown 1000:1000 .secrets/api-key
```

It is recommended to change the permission so only the owner can read/write and nobody else can access the secret.
In the example above we changed the owner and group to 1000 which means the secret will be readable by the default user in the VM.

After launching a VM, the secrets will be available in the guest VM filesystem at `/run/slicer/secrets`.

Laucnh a VM with the secrets from the example and list secrets:

```bash
ls -l /run/slicer/secrets/
```

Should give an output similar to:

```
total 4
-rw------- 1 ubuntu ubuntu 11 Dec  8 11:12 api-key
```

## Managing Secrets with the CLI

The `slicer secret` command provides a complete interface for managing secrets.

> Note that the Slicer API needs to be running to manage secrets using the CLI.

### Create a secret

To create a secret from stdin run:

```bash
echo "my secret value" | slicer secret create my-secret
```

Or pipe the value from a file instead:

```bash
cat /path/to/secret.txt | slicer secret create my-secret
```

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

Secrets can be assigned to specific users and groups using the `--uid` and `--gid` flags and permissions can be changed using the `--permissions` flag. This is useful when secrets need to be accessible by specific system users or services within the guest VM.

By default secrets created using the CLI are owned by root and have permissions set to 0600.

### List secrets

View all secrets:

```bash
slicer secret list
```

Output:

```
NAME
----
my-secret
```

Use the `-v` flag to view more details like permissions and ownership:

```bash
slicer secret list -v
```

Output:

```
NAME                      SIZE       PERMISSIONS  UID      GID
----                      ----       -----------  ---      ---
my-secret                 11         0600         0        0
```

### Update a secret

Update a secret from a file:

```bash
slicer secret update my-secret --from-file=/path/to/newsecret.txt
```

Update permissions and ownership:

```bash
slicer secret update my-secret \
  --permissions=0644 \
  --uid=1001 \
  --gid=1001
```

### Remove a secret

Remove a secret by name:

```bash
slicer secret remove my-secret
```

## Scoped secret access

You might not always want to sync all secrets to every VM. Slicer allows you to select which secrets should be synced when launching a VM using the CLI.

For example, to make only the `my-secret` secret available but no other secrets run:

```bash
slicer vm launch --secret=my-secret
```

## Sync secrets after VM boot.

If secrets have been added, updated or removed after a VM has been started, they can be re-synced by running the following command in the Slicer VM:

```bash
sudo slicer-agent secrets sync
```

## Guest VM Requirements

The `slicer-agent` daemon must be running within the guest VM to support secrets. This is included as a default in the various supported images.
