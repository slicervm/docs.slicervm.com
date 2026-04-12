# Slicer Platform

Slicer gives you on-demand Linux VMs through a REST API and Go SDK. You can create a VM, run commands inside it, copy files in and out, and delete it - all over HTTP.

This section assumes [Slicer for Linux](/getting-started/install/), but many of the REST API examples will also work on [Slicer for Mac](/mac/overview/) if you use the `sbox` host group.

By installing and starting Slicer, you agree to the [End User License Agreement (EULA)](https://slicervm.com/eula/).

## Self-hosted, real microVMs

Slicer runs on your hardware. Every VM is a real microVM with its own kernel - not a container dressed up as one. Unlike hosted platforms like Modal, Daytona, or Fly, there are no artificial timeouts, no per-second metering, no rate limits on the API, and no mandatory scale-to-zero. Your VMs run for as long as you need them, using as many resources as the host has available.

Your data never leaves your network, there is no third-party control plane, and you pay a flat rate per [Platform license](https://slicervm.com/pricing/) regardless of how many VMs you run.

## What you get

Each VM runs a real Linux kernel with systemd. It is not a container.

* Full OS with package managers, cron, and systemd services
* Dedicated kernel - no shared kernel attack surface
* Network isolation between VMs
* Sub-second cold boot with [ZFS](/storage/zfs) or [devmapper](/storage/devmapper) storage

VMs launched through the API are called **sandboxes**. They are ephemeral by default: shut down Slicer and they are cleaned up automatically. Sandboxes can also be configured to persist, which is how multi-tenant platforms and the [Kubernetes autoscaler](/examples/autoscaling-k3s/) work - VMs stay running until you explicitly delete them.

To create a persistent sandbox, pass `"persistent": true` in the create request (or `--persistent` via the CLI). Persistent VMs survive Slicer restarts and their disk is retained until you delete the VM.

## Common use-cases

* **Code execution**: run untrusted or user-submitted code in a VM that gets destroyed afterwards
* **AI agents**: give a coding agent root access to a disposable Linux environment
* **CI/CD tasks**: compile, test, and package in isolated VMs without contaminating shared runners
* **Batch processing**: spin up VMs to process files, convert media, or run data pipelines, then tear them down
* **Serverless tasks**: ETL, video processing, encryption, file scanning, handling sensitive data - tasks that could run on AWS Lambda or OpenFaaS, but suit a full OS with root access better
* **On-demand dev environments**: provision a full Linux environment per pull request or per developer

## How it works

1. Define a host group in YAML with `count: 0` - no VMs are pre-allocated
2. Start Slicer with `slicer up`
3. Create sandboxes on demand through the API
4. Run commands, copy files, read output
5. Delete the VM when done

The host group sets defaults (CPU, RAM, image, storage backend). Individual VMs can override CPU and RAM at creation time.

VM hostnames are auto-assigned with an incrementing integer based on the host group name (e.g. `sandbox-1`, `sandbox-2`, `sandbox-3`). If you need to correlate a VM with a user, job, or request in your system, pass `tags` when creating the VM. Tags are returned in list responses and can be used to look up VMs later.

## Reference architecture

The difference between "Slicer for Linux" and "Slicer Platform" is who is driving. With Slicer for Linux, a person runs CLI commands. With Slicer Platform, your application drives Slicer through the API.

```
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  Your Users  │ ────►  │  Your App    │ ────►  │  Slicer API  │
└──────────────┘        └──────────────┘        └──────┬───────┘
                                                       │
                                              create / exec / cp / delete
                                                       │
                                                       ▼
                                                ┌──────────────┐
                                                │   microVMs   │
                                                └──────────────┘
```

There are two deployment models depending on your isolation requirements:

* [Single Slicer instance](/platform/single-instance/) - one daemon, all tenants share it, use tags to track ownership
* [Instance per tenant](/platform/instance-per-tenant/) - one daemon per tenant with its own UNIX socket and isolated network

A typical request flow for a code execution platform:

1. User submits code through your frontend
2. Your backend calls `POST /hostgroup/sandbox/nodes` to create a VM
3. Poll `GET /vm/HOSTNAME/health` until the agent is ready
4. `POST /vm/HOSTNAME/cp` to copy the code into the VM
5. `POST /vm/HOSTNAME/exec` to run it
6. `GET /vm/HOSTNAME/cp` to copy results back
7. `DELETE /hostgroup/sandbox/nodes/HOSTNAME` to clean up
8. Return the result to the user

The entire flow takes seconds. Each user gets a dedicated VM with its own kernel - no shared state, no container escapes.

## API surfaces

Slicer exposes three ways to manage VMs programmatically:

* **REST API**: HTTP endpoints for the full VM lifecycle. Works from any language. See the [API reference](/reference/api/).
* **Go SDK**: a Go client library that wraps the REST API. See the [SDK on GitHub](https://github.com/slicervm/sdk).
* **CLI**: `slicer vm` commands that call the same API, useful for scripting and exploration.

## Next steps

* [Quickstart: create a sandbox via API](/platform/quickstart/) - working curl examples in under 3 minutes
* [REST API reference](/reference/api/) - full endpoint documentation
* [Go SDK](https://github.com/slicervm/sdk) - programmatic access from Go
* [Authentication and networking](/reference/api/#authentication) - securing the API for production
* [Storage backends](/storage/overview/) - choosing between image, ZFS, and devmapper
