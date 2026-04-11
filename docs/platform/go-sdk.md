# Go SDK

The [Slicer Go SDK](https://github.com/slicervm/sdk) wraps the REST API into a typed Go client. It covers the full VM lifecycle: creating and deleting VMs, executing commands, copying files, managing secrets, and polling for readiness.

```bash
go get github.com/slicervm/sdk@latest
```

## Examples

The [SDK repository](https://github.com/slicervm/sdk) includes working examples. Run any of them with `go run ./examples/NAME` after setting `SLICER_URL` and `SLICER_TOKEN`.

* [Video conversion](/platform/video-conversion/) - full walkthrough: create a VM, install ffmpeg, copy files in/out, convert video, clean up
* [Create a VM](https://github.com/slicervm/sdk/tree/main/examples/create) - minimal VM creation with host group defaults
* [File transfer](https://github.com/slicervm/sdk/tree/main/examples/transform) - copy a file into a VM, process it with `CommandContext`, copy the result back
* [Headless Claude Code](https://github.com/slicervm/sdk/tree/main/examples/claude) - run Claude Code in an isolated VM with credential forwarding from macOS
* [K3s via userdata](https://github.com/slicervm/sdk/tree/main/examples/k3s-userdata) - bootstrap a Kubernetes cluster in a VM and extract the kubeconfig

## Connecting to the API

The SDK client takes a base URL, bearer token, and user agent string:

```go
import slicer "github.com/slicervm/sdk"

client := slicer.NewSlicerClient(
    "http://127.0.0.1:8080",
    os.Getenv("SLICER_TOKEN"),
    "my-app/1.0",
    nil,
)
```

The token is saved at `/var/lib/slicer/auth/token` when API authentication is enabled. See [authentication](/reference/api/#authentication) for setup.

The SDK also supports UNIX sockets. Pass the socket path instead of a URL:

```go
client := slicer.NewSlicerClient("/var/run/slicer/api.sock", token, "my-app/1.0", nil)
```

## Create a VM

Create a VM in a host group. Pass an empty request to use the host group defaults, or override CPU and RAM:

```go
ctx := context.Background()

node, err := client.CreateVM(ctx, "sandbox", slicer.SlicerCreateNodeRequest{})
if err != nil {
    log.Fatal(err)
}

fmt.Printf("hostname=%s ip=%s\n", node.Hostname, node.IP)
```

To override resources or pass a boot script:

```go
node, err := client.CreateVM(ctx, "sandbox", slicer.SlicerCreateNodeRequest{
    RamBytes: 4 * 1024 * 1024 * 1024,
    CPUs:     4,
    Userdata: "#!/bin/bash\napt update -qy && apt install -qy curl",
})
```

## Wait for the agent

The guest agent starts after the VM boots. Poll until it responds before running commands or copying files:

```go
for attempt := 1; attempt <= 30; attempt++ {
    _, err := client.GetAgentHealth(ctx, node.Hostname, false)
    if err == nil {
        break
    }
    time.Sleep(250 * time.Millisecond)
}
```

If you passed userdata and need to wait for the script to finish, check the `UserdataRan` field:

```go
for {
    health, err := client.GetAgentHealth(ctx, node.Hostname, true)
    if err == nil && health.UserdataRan {
        break
    }
    time.Sleep(250 * time.Millisecond)
}
```

## Run commands

The SDK provides two ways to execute commands. `CommandContext` follows the `os/exec` pattern:

```go
cmd := client.CommandContext(ctx, node.Hostname, "uname", "-a")
cmd.UID = 1000
cmd.GID = 1000

out, err := cmd.Output()
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(out))
```

For streaming output, use `Exec` which returns a channel:

```go
ch, err := client.Exec(ctx, node.Hostname, slicer.SlicerExecRequest{
    Command: "apt update && apt install -y nginx",
    Shell:   "/bin/bash",
})
if err != nil {
    log.Fatal(err)
}

for res := range ch {
    if res.Error != "" {
        log.Fatal(res.Error)
    }
    fmt.Print(res.Stdout)
}
```

## Copy files

Upload a file to the VM:

```go
err := client.CpToVM(ctx, node.Hostname,
    "./input.txt",           // local path
    "/home/ubuntu/input.txt", // VM path
    1000, 1000,              // uid, gid
    "0644",                  // permissions
    "binary",                // mode: "binary" or "tar"
)
```

Download a file from the VM:

```go
err := client.CpFromVM(ctx, node.Hostname,
    "/home/ubuntu/output.txt", // VM path
    "./output.txt",            // local path
    "0644",                    // permissions
    "binary",                  // mode
)
```

Use `"tar"` mode for directories. See [copy files](/reference/api/#copy-files-to-and-from-the-microvm) in the API reference.

## Delete a VM

```go
_, err := client.DeleteVM(ctx, "sandbox", node.Hostname)
```

Use `defer` to clean up VMs on program exit:

```go
defer func() {
    client.DeleteVM(context.Background(), "sandbox", node.Hostname)
}()
```

## Pause and resume

Pause a VM to free CPU while keeping its state in memory, then resume when needed:

```go
err := client.PauseVM(ctx, node.Hostname)
// later...
err = client.ResumeVM(ctx, node.Hostname)
```

## Method reference

### VM operations

| Method | Description |
|--------|-------------|
| `CreateVM(ctx, groupName, request)` | Create a VM in a host group |
| `DeleteVM(ctx, groupName, hostname)` | Delete a VM |
| `ListVMs(ctx)` | List all VMs across host groups |
| `GetHostGroups(ctx)` | List host groups |
| `GetHostGroupNodes(ctx, groupName)` | List VMs in a host group |
| `PauseVM(ctx, hostname)` | Pause a running VM |
| `ResumeVM(ctx, hostname)` | Resume a paused VM |
| `Shutdown(ctx, hostname, request)` | Shutdown or reboot a VM |
| `GetVMStats(ctx, hostname)` | Get CPU, memory, and disk stats |
| `GetVMLogs(ctx, hostname, lines)` | Get serial console logs |

### Guest operations

| Method | Description |
|--------|-------------|
| `CommandContext(ctx, hostname, cmd, args...)` | Run a command (`os/exec`-style interface) |
| `Exec(ctx, hostname, request)` | Execute a command with streaming output |
| `CpToVM(ctx, vmName, localPath, vmPath, uid, gid, perms, mode)` | Upload a file or directory |
| `CpFromVM(ctx, vmName, vmPath, localPath, perms, mode)` | Download a file or directory |
| `GetAgentHealth(ctx, hostname, includeStats)` | Check agent readiness |

### Secret management

| Method | Description |
|--------|-------------|
| `CreateSecret(ctx, request)` | Create a secret |
| `ListSecrets(ctx)` | List secrets (metadata only) |
| `PatchSecret(ctx, name, request)` | Update a secret |
| `DeleteSecret(ctx, name)` | Delete a secret |

## See also

* [SDK source on GitHub](https://github.com/slicervm/sdk)
* [REST API reference](/reference/api/) - the underlying HTTP endpoints
* [Quickstart](/platform/quickstart/) - curl-based walkthrough of the same lifecycle
* [Networking](/reference/networking/) - CIDR ranges and routing
