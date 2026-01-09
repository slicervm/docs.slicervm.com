# Execute Commands in VM via SDK

The [Slicer SDK](https://github.com/slicervm/sdk) for Go enables programmatic management of Slicer VMs. It supports the full VM lifecycle, including creation, deletion of VMs,and provides methods for file transfer and executing commands within the VM. This allows you to automate Slicer to run ephemeral or isolated workloads and retrieve the results efficiently.

Some common use cases could include:

* Running AI agents or inference workloads
* Executing cluster nodes (Kubernetes, Nomad, etc.)
* Building or running untrusted customer code in isolation
* Media conversion and metadata extraction
* Headless browser automation
* Batch workloads with dynamic scaling

## Example: Video Conversion Workflow

In this example we are going to use the SDK to provision a short-lived Linux VM, install `ffmpeg`, and use it to convert a mkv video to MP4. After the conversion is done the result is copied out of the VM before it is teared down.

### Run slicer
 
Before creating and running the SDK example script ensure that a Slicer API server is running.

Create a basic Slicer configuration using `slicer new`.
```bash
slicer new sdk \
  --count=0 \
  --graceful-shutdown=false \
  > sdk.yaml
```

By using the `--count=0` flag we create a host group with no pre‑allocated VMs, the SDK will be used to provision VMs on demand. Disabling graceful shutdown reduces teardown latency for short‑lived workloads. In this example we also set the flag `--persistent=false` to ensure the VM storage is always cleared after the VM is destroyed.

For the fastest possible boot times, [use ZFS for storage](/storage/zfs).

If you have ZFS set up, you can simply replace the storage flags with something like:

```bash
--storage=zfs
```

Start the Slicer API using the generated configuration:

```bash
sudo -E slicer up sdk.yaml
```

### Create a sample program

#### Step 1: Initialize SDK client and create a VM

The SDK client handles authentication and communication with the SlicerVM API. VM resource requirements such as CPU count and memory are specified at creation time.
```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	sdk "github.com/slicervm/sdk"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	// Initialize SDK client with authentication
	baseURL := os.Getenv("SLICER_URL")
	token := os.Getenv("SLICER_TOKEN")
	userAgent := "video-converter/1.0"

	client := sdk.NewSlicerClient(baseURL, token, userAgent, nil)
	ctx := context.Background()

	// Define VM specifications
	createReq := sdk.SlicerCreateNodeRequest{
		RamBytes:    8 * 1024 * 1024 * 1024, // 8GB RAM
		CPUs:        4, // 4 CPU cores for video processing
	}

	// Create VM in the 'sdk' host group
	hostGroupName := "sdk"
	node, err := client.CreateNode(ctx, hostGroupName, createReq)
	if err != nil {
		return fmt.Errorf("failed to create VM: %s", err)
	}

	log.Printf("Created VM: %s", node.Hostname)
	
	// Ensure VM cleanup on program exit
	defer func() {
		log.Printf("Cleaning up VM: %s", node.Hostname)
		if err := client.DeleteNode(hostGroupName, node.Hostname); err != nil {
			log.Printf("Failed to delete VM: %s", err)
		}
	}()
}
```

#### Step 2: Wait for agent readiness

After the VM is created, the SlicerVM agent must be initialized before commands or file transfers can be executed. Agent readiness should always be verified before trying to execute commands or transfer files.

```go
	// Wait for SlicerVM agent to become ready
	log.Println("Waiting for slicer agent to initialize...")

	agentReady := false
	for attempt := 1; attempt <= 30; attempt++ {
		_, err := client.GetAgentHealth(ctx, node.Hostname, false)
		if err == nil {
			agentReady = true
			log.Println("Slicer agent is ready")
			break
		}

		log.Printf("Agent not ready (attempt %d/30): %s", attempt, err)
		time.Sleep(100 * time.Millisecond)
	}

	if !agentReady {
		return fmt.Errorf("timeout waiting for slicer agent to become ready")
	}
```

#### Step 3: Install required packages

Use `Exec()` to install ffmpeg into the VM.

```go
	// Install ffmpeg for video processing
	log.Println("Installing ffmpeg...")

	installCmd, err := client.Exec(ctx, node.Hostname, sdk.SlicerExecRequest{
		Command: "apt update && apt install -y ffmpeg",
		Shell:   "/bin/bash",
	})
	if err != nil {
		return fmt.Errorf("failed to start ffmpeg installation: %s", err)
	}

	// Process command output in real-time
	for response := range installCmd {
		if len(response.Error) > 0 {
			return fmt.Errorf("ffmpeg installation failed: %s", response.Error)
		}

		// Log command output
		if response.Stdout != "" {
			log.Printf("[install ffmpeg] %s", response.Stdout)
		}
		if response.Stderr != "" {
			log.Printf("[install ffmpeg] %s", response.Stderr)
		}

		// Check final exit status
		if response.ExitCode != 0 {
			return fmt.Errorf("ffmpeg installation failed with exit code: %d", response.ExitCode)
		}
	}
```

To speed up the workflow, instead of installing required packages each time, you could derive a custom image with ffmpeg pre-installed. See [Build a custom image](/tasks/custom-image/) for more info.

#### Step 4: Transfer files and run Exec

An input video file is copied into the VM, processed with ffmpeg, and then the result is copied back out. The SDK has a `CpToVM()` and `CpFromVM()` method for file transfer. File ownership and permissions can be explicitly controlled. Commands can be executed inside the VM using the `Exec()` method. Output is streamed in real time, allowing progress monitoring and failure detection.


```go
// Transfer input file to VM
	log.Println("Copying input file to VM...")
	err = client.CpToVM(ctx, node.Hostname, "./input.mkv", "/home/ubuntu/input.mkv", 1000, 1000, "0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy input file: %s", err)
	}

	// Execute video conversion
	log.Println("Converting video...")

	convertCmd, err := client.Exec(ctx, node.Hostname, sdk.SlicerExecRequest{
		Cwd:     "/home/ubuntu",
		Command: "ffmpeg -i input.mkv -vf scale=-2:720 -c:v libx264 -preset medium -crf 23 -c:a aac output.mp4",
		Shell:   "/bin/bash",
		UID:     1000,
		GID:     1000,
	})
	if err != nil {
		return fmt.Errorf("failed to start conversion: %s", err)
	}

	// Monitor conversion progress
	for response := range convertCmd {
		if len(response.Error) > 0 {
			return fmt.Errorf("conversion failed: %s", response.Error)
		}

		if response.Stdout != "" {
			log.Printf("[ffmpeg] %s", response.Stdout)
		}

		if response.Stderr != "" {
			log.Printf("[ffmpeg] %s", response.Stderr)
		}

		if response.ExitCode != 0 {
			return fmt.Errorf("conversion failed with exit code: %d", response.ExitCode)
		}
	}

	// Copy converted file back to host
	log.Println("Copying result back to host...")
	err = client.CpFromVM(ctx, node.Hostname, "/home/ubuntu/output.mp4", "./output.mp4", "0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy output file: %s", err)
	}

	log.Println("Video conversion completed successfully!")
```

After copying in the source file we execute ffmpeg on the VM to convert the `mkv` file into an `mp4` file.

```bash
ffmpeg -i input.mkv -vf scale=-2:720 -c:v libx264 -preset medium -crf 23 -c:a aac output.mp
```

* `-i input.mkv` – input video file
* `-vf scale=-2:720` – scales video to 720px height while keeping aspect ratio (-2 auto-calculates width, divisible by 2)
* `-c:v libx264` – encodes video using H.264
* `-preset medium` – balances encoding speed and compression efficiency
* `-crf 23` – sets video quality (lower = better quality, larger file)
* `-c:a aac` – encodes audio using AAC
* `output.mp4` – output file in MP4 format

The input file used in this example can be found in the [GitHub repository](https://github.com/welteki/slicer-sdk-example).

In step 1 we used a `defer` to delete the VM via the API after a conversion is complete.

It is also possible to trigger a shutdown from within the VM by executing the `sudo reboot` command. If the VM has persistent storage turned off the disk is automatically deleted after the VM is shutdown.

### Run the example

Configure environment variables for our example program. These are used to connect to the Slicer API. You can connect to a local slicer instance running on the same host, like what we are doing here, or to a remote instance.

```bash
export SLICER_URL="http://127.0.0.1:8080"
export SLICER_TOKEN="$(sudo cat /var/lib/slicer/auth/token)"
```

If you are running the slicer API on a remote host you could get the token over SSH:

```bash
export SLICER_TOKEN="$(ssh user@remote-host 'sudo cat /var/lib/slicer/auth/token')"
```

Download an example input video file:

```bash
curl -L -o input.mkv https://github.com/welteki/slicer-sdk-example/raw/refs/heads/main/input.mkv
```

Full `main.go` file:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	sdk "github.com/slicervm/sdk"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	// Initialize SDK client with authentication
	baseURL := os.Getenv("SLICER_URL")
	token := os.Getenv("SLICER_TOKEN")
	userAgent := "video-converter/1.0"

	client := sdk.NewSlicerClient(baseURL, token, userAgent, nil)
	ctx := context.Background()

	// Define VM specifications
	createReq := sdk.SlicerCreateNodeRequest{
		RamGB:    8, // 8GB RAM
		CPUs:     4, // 4 CPU cores for video processing
	}

	// Create VM in the 'sdk' host group
	hostGroupName := "sdk"
	node, err := client.CreateNode(ctx, hostGroupName, createReq)
	if err != nil {
		return fmt.Errorf("failed to create VM: %s", err)
	}

	log.Printf("Created VM: %s", node.Hostname)

	// Ensure VM cleanup on program exit
	defer func() {
		log.Printf("Cleaning up VM: %s", node.Hostname)
		if err := client.DeleteNode(hostGroupName, node.Hostname); err != nil {
			log.Printf("Failed to delete VM: %s", err)
		}
	}()

	// Wait for SlicerVM agent to become ready
	log.Println("Waiting for slicer agent to initialize...")

	agentReady := false
	for attempt := 1; attempt <= 30; attempt++ {
		_, err := client.GetAgentHealth(ctx, node.Hostname, false)
		if err == nil {
			agentReady = true
			log.Println("Slicer agent is ready")
			break
		}

		log.Printf("Agent not ready (attempt %d/30): %s", attempt, err)
		time.Sleep(100 * time.Millisecond)
	}

	if !agentReady {
		return fmt.Errorf("timeout waiting for slicer agent to become ready")
	}

	// Install ffmpeg for video processing
	log.Println("Installing ffmpeg...")

	installCmd, err := client.Exec(ctx, node.Hostname, sdk.SlicerExecRequest{
		Command: "apt update && apt install -y ffmpeg",
		Shell:   "/bin/bash",
	})
	if err != nil {
		return fmt.Errorf("failed to start ffmpeg installation: %s", err)
	}

	// Process command output in real-time
	for response := range installCmd {
		if len(response.Error) > 0 {
			return fmt.Errorf("ffmpeg installation failed: %s", response.Error)
		}

		// Log command output
		if response.Stdout != "" {
			log.Printf("[install ffmpeg] %s", response.Stdout)
		}
		if response.Stderr != "" {
			log.Printf("[install ffmpeg] %s", response.Stderr)
		}

		// Check final exit status
		if response.ExitCode != 0 {
			return fmt.Errorf("ffmpeg installation failed with exit code: %d", response.ExitCode)
		}
	}

	// Transfer input file to VM
	log.Println("Copying input file to VM...")
	err = client.CpToVM(ctx, node.Hostname, "./input.mkv", "/home/ubuntu/input.mkv", 1000, 1000, "0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy input file: %s", err)
	}

	// Execute video conversion
	log.Println("Converting video...")

	convertCmd, err := client.Exec(ctx, node.Hostname, sdk.SlicerExecRequest{
		Cwd:     "/home/ubuntu",
		Command: "ffmpeg -i input.mkv -vf scale=-2:720 -c:v libx264 -preset medium -crf 23 -c:a aac output.mp4",
		Shell:   "/bin/bash",
		UID:     1000,
		GID:     1000,
	})
	if err != nil {
		return fmt.Errorf("failed to start conversion: %s", err)
	}

	// Monitor conversion progress
	for response := range convertCmd {
		if len(response.Error) > 0 {
			return fmt.Errorf("conversion failed: %s", response.Error)
		}

		if response.Stdout != "" {
			log.Printf("[ffmpeg] %s", response.Stdout)
		}

		if response.Stderr != "" {
			log.Printf("[ffmpeg] %s", response.Stderr)
		}

		if response.ExitCode != 0 {
			return fmt.Errorf("conversion failed with exit code: %d", response.ExitCode)
		}
	}

	// Copy converted file back to host
	log.Println("Copying result back to host...")
	err = client.CpFromVM(ctx, node.Hostname, "/home/ubuntu/output.mp4", "./output.mp4", "0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy output file: %s", err)
	}

	log.Println("Video conversion completed successfully!")
	return nil
}
```

> The full code sample is also available on [GitHub](https://github.com/welteki/slicer-sdk-example/tree/main)

Run the program:

```bash
go run main.go
```

In this example we created an ephemeral VM that is destroyed after the file conversion completes and the result is extracted. However it is entirely up to your implementation how you want to handle the VM lifecycle strategy.

Possible VM lifecycle strategies:

* **Single use**: Create a VM per task and destroy it immediately after completion (as shown in this example). Maximizes isolation.
* **Reusable VMs**: Reuse a VM to execute multiple tasks in sequence or in parallel to reduce startup overhead.

## Next Steps

* Review the [SDK reference](https://github.com/slicervm/sdk)
* Explore the [REST API documentation](/reference/api/) for direct integrations
