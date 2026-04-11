# Example: Video Conversion

This example uses the Go SDK to create a VM, install ffmpeg, copy a video file in, convert it, and copy the result back. The VM is deleted when the program exits.

For the SDK reference and other examples, see the [Go SDK](/platform/go-sdk/) page.

## Set up Slicer

Create a host group with no pre-allocated VMs:

```bash
slicer new sdk \
  --count=0 \
  --graceful-shutdown=false \
  > sdk.yaml

sudo slicer up sdk.yaml
```

Set the connection details:

```bash
export SLICER_URL="http://127.0.0.1:8080"
export SLICER_TOKEN="$(sudo cat /var/lib/slicer/auth/token)"
```

## The program

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
	client := sdk.NewSlicerClient(
		os.Getenv("SLICER_URL"),
		os.Getenv("SLICER_TOKEN"),
		"video-converter/1.0",
		nil,
	)
	ctx := context.Background()

	node, err := client.CreateVM(ctx, "sdk", sdk.SlicerCreateNodeRequest{
		RamBytes: 8 * 1024 * 1024 * 1024,
		CPUs:     4,
	})
	if err != nil {
		return fmt.Errorf("failed to create VM: %s", err)
	}
	log.Printf("Created VM: %s", node.Hostname)

	defer func() {
		log.Printf("Cleaning up VM: %s", node.Hostname)
		client.DeleteNode("sdk", node.Hostname)
	}()

	// Wait for agent
	for attempt := 1; attempt <= 30; attempt++ {
		if _, err := client.GetAgentHealth(ctx, node.Hostname, false); err == nil {
			break
		}
		time.Sleep(100 * time.Millisecond)
	}

	// Install ffmpeg
	installCmd, err := client.Exec(ctx, node.Hostname, sdk.SlicerExecRequest{
		Command: "apt update && apt install -y ffmpeg",
		Shell:   "/bin/bash",
	})
	if err != nil {
		return fmt.Errorf("failed to install ffmpeg: %s", err)
	}
	for response := range installCmd {
		if response.Error != "" {
			return fmt.Errorf("install failed: %s", response.Error)
		}
	}

	// Copy input file to VM
	err = client.CpToVM(ctx, node.Hostname,
		"./input.mkv", "/home/ubuntu/input.mkv",
		1000, 1000, "0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy input: %s", err)
	}

	// Run conversion
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
	for response := range convertCmd {
		if response.Error != "" {
			return fmt.Errorf("conversion failed: %s", response.Error)
		}
		if response.Stderr != "" {
			log.Printf("[ffmpeg] %s", response.Stderr)
		}
	}

	// Copy result back
	err = client.CpFromVM(ctx, node.Hostname,
		"/home/ubuntu/output.mp4", "./output.mp4",
		"0664", "binary")
	if err != nil {
		return fmt.Errorf("failed to copy output: %s", err)
	}

	log.Println("Video conversion completed")
	return nil
}
```

Run with:

```bash
go run main.go
```

To speed things up, [build a custom image](/platform/custom-images/) with ffmpeg pre-installed so you skip the `apt install` on every run.

## See also

* [Go SDK](/platform/go-sdk/) - SDK reference and other examples
* [Build a custom image](/platform/custom-images/) - bake dependencies into the root filesystem
* [REST API reference](/reference/api/) - the underlying HTTP endpoints
