# Introduction to Slicer

SlicerVM gives you **real Linux, in milliseconds**.

Full VMs with systemd and a real kernel, on your Mac, your servers, or your cloud.
Slicer is built for teams that need isolation and control without moving code and data to third-party infrastructure.

## Where Slicer fits

Slicer is useful for both one-off workloads and long-running Linux workloads.

* **Sandboxes** — API-driven, disposable environments for short-lived execution and automation.
* **Services** — persistent VMs where you need stable hosts and ongoing operations.

Both are managed through the same CLI/API/SDK surfaces.

In both modes you get full Linux behavior where containers are not enough: package managers, services, networking, and predictable startup.

## Our own use-cases

Slicer was developed internally at OpenFaaS Ltd and has been used across product, platform, and support work.

* Booting Kubernetes clusters for Helm chart and platform validation in minutes.
* Reproducing customer issues quickly in disposable lab environments.
* Running the code review bot in isolated microVMs for safer, repeatable review flows.
* Running chaos and failure-mode testing with network and execution control.
* Building AI/LLM workflows in isolated Linux environments.
* Reducing R&D cost and latency by replacing slower cloud VMs with local mini PCs, NUCs, and bare-metal hosts.
* Replacing Docker Desktop/Lima/UTM on macOS with a local-first Linux workflow.
* Creating quick demo environments for customer trials to reduce friction during B2B evaluations.

## Conceptual architecture

![Slicer architecture](/images/slicer-arch.svg)

In Slicer:

* On Linux, it uses KVM with Firecracker by default.
* Use Cloud Hypervisor when you need PCI device passthrough.
* On macOS, Slicer for Mac runs on Apple’s Virtualization Framework.
* Mac guests can use VirtioFS folder sharing for local paths.
* Mac guests support Rosetta for x86_64 Linux binaries.
* You can start Slicer with a fixed host-group count or zero hosts and create VMs on demand through the API.
* Slicer runs as a daemon and exposes API, CLI, and SDK management for microVM host groups.
* You define host groups and VM specs in YAML before launching VMs.
* The `slicer-agent` enables file copy, command execution, secret sync, and serial shell access inside each guest.
* API-first workflows remain local-first for low-latency execution and data paths.

## Where you can run it

Slicer runs on Linux, including x86_64 and arm64 systems, and on Apple Silicon via Slicer for Mac.

If you need local Linux on macOS, check out [Slicer for Mac](/mac/overview).

If you need production-hosted Linux microVMs, Slicer works well on bare-metal and in nested setups on major cloud providers.
You can also run it on WSL2 for local experimentation, labs, and home office use.

## Next steps

- [Install Slicer on Linux](/getting-started/install)
- [Install Slicer for Mac](/mac/installation)
- [Try the walkthrough](/getting-started/walkthrough)
- [Run a one-shot task](/examples/run-a-task/)
- [GPU workloads with Slicer](/examples/gpu-ollama/)
- [Get in touch](/contact) for commercial questions
