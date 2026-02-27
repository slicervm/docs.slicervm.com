# Introduction to Slicer

SlicerVM gives you **real Linux, in milliseconds**.

Full VMs with systemd and a real kernel, on your Mac, your servers, or your cloud.
Slicer is built for teams that need isolation and control without moving code and data to third-party infrastructure.

## Where Slicer fits

Slicer is useful for both one-off/ephemeral workloads and long-running Linux services.

All Slicer VMs are called VMs both in the YAML, the API, and codebase, however, we talk about them in two different ways:

* **Sandboxes** — API-driven, disposable environments for short-lived execution and automation. These VMs are launched into a host group which has a `count: 0` on start-up.
* **Services** — persistent VMs where you need stable hosts for hosting, long-term testing, etc. These VMs are launched from a host group which has a `count: 1` or higher.

Both are managed through the same CLI/API/SDK surfaces.

In both modes you get a full Linux system where often containers are not enough: package managers, services that need to run as root, bespoke networking configurations, and predictable startup/tear-down times.

Example of Slicer used for a Service: [A remote (persistent) BuildKit daemon](/examples/buildkit/) or [HA K3s cluster](/examples/ha-k3s/)

Example of a Slicer used as a Sandbox: [Run a task](/examples/run-a-task/) or [Process video files with ffmpeg](/tasks/execute-commands-with-sdk/)

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

Slicer for Linux

* On Linux, KVM is used with [Firecracker](https://firecracker-microvm.github.io/) by default. [Cloud Hypervisor](https://www.cloudhypervisor.org/) is an option when you need PCI passthrough for devices like NICs or GPUs.
* You can start Slicer with a fixed host-group count or zero hosts and create VMs on demand through the API.

Slicer for Mac:

* On macOS, Slicer for Mac uses [Apple’s Virtualization Framework](https://developer.apple.com/documentation/virtualization).
* Mac guests can use VirtioFS folder sharing for local paths.
* Mac guests support Rosetta for x86_64 Linux binaries.

* Slicer runs as a daemon and exposes API, CLI, and SDK management for microVM host groups.
* You define host groups and VM specs in YAML before launching VMs. These cannot be created via API at this point in time. Host groups need to have [non-overlapping networking](/reference/networking/) CIDRs defined.
* The guest agent `slicer-agent` enables deep integration without having to use networking for: file copies, command execution, secret syncing, shutdowns, and direct shell access to the VM.

## Where you can run it

Slicer runs on Linux, including x86_64 and arm64 systems, and on Apple Silicon via Slicer for Mac.

If you need local Linux on macOS, check out [Slicer for Mac](/mac/overview).

If you need production-hosted Linux microVMs, Slicer works well on bare-metal and in nested virtualisation on cloud providers.

You can also run it on WSL2 for local experimentation, labs, and home office use.

## Next steps

- [Install Slicer on Linux](/getting-started/install)
- [Install Slicer for Mac](/mac/installation)
- [Try the walkthrough](/getting-started/walkthrough)
- [Run a one-shot task](/examples/run-a-task/)
- [GPU workloads with Slicer](/examples/gpu-ollama/)
- [Get in touch](/contact) for commercial questions
