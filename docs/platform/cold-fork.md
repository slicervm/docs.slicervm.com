# Build once, fork many

Cold forking starts new microVMs from the disk state of a stopped, persistent VM.

Use it when the VM needs an expensive setup step before each job: installing a compiler, downloading dependencies, pulling container images, or preparing an agent. Set up one builder, commit its disk, then fork as many clean children as you need.

Only disk state is committed. Each child boots normally with its own hostname, machine ID, and network identity.

## Requirements

Cold fork currently requires:

* Firecracker
* `image`, `devmapper`, or `zfs` storage
* Bridge or isolated networking
* A stopped, persistent source VM

Network overrides on `slicer vm fork`, including `--allow`, `--no-allow`, and `--drop`, require isolated networking. Cold forking itself also works with bridge networking, but the child inherits the host group's network policy unchanged.

Forked children are persistent. Delete them explicitly when the job is complete.

## Create an empty host group

This example uses the min image for the fastest boot, image storage for the simplest setup, and isolated networking so the fork can have its egress removed:

```bash
slicer new runners \
  --count 0 \
  --min \
  --storage image \
  --net isolated \
  > slicer.yaml
```

Start Slicer:

```bash
sudo -E slicer up slicer.yaml
```

The generated isolated host group allows egress by default. The builder can download packages and dependencies before its disk is committed.

## Prepare the builder

In another terminal, create a persistent VM:

```bash
BUILDER=$(slicer vm add runners \
  --persistent \
  --tag purpose=cold-fork-builder \
  --wait \
  --timeout 10m \
  --json | jq -r '.hostname')

echo "$BUILDER"
```

The response contains the allocated hostname. Install and prepare everything that should be present in each child. For example:

```bash
slicer vm exec "$BUILDER" -- \
  "sudo arkade system install go"

slicer vm exec "$BUILDER" -- \
  "git clone https://github.com/alexellis/arkade \
     /home/ubuntu/arkade && \
   cd /home/ubuntu/arkade && \
   /usr/local/go/bin/go build -mod=vendor -o ./arkade"
```

Do not store reusable credentials in the builder. Every fork inherits its disk.

## Commit the disk

Shut down the builder, then commit it:

```bash
slicer vm shutdown "$BUILDER"

# The Firecracker shutdown request returns after the VM process has stopped.

COMMIT=$(slicer vm commit "$BUILDER" \
  --tag arkade \
  --label toolchain=go \
  --cache-key arkade-go-v1 \
  --quiet)
echo "$COMMIT"
```

The commit ID identifies an immutable disk parent, for example:

```text
cmt-runnersx1-a1b2c3d4e5f6a7b8
```

The source VM remains stopped. A committed source cannot be committed again with different metadata; create another persistent builder when you need a different base.

## Fork a child

To inherit the host group's existing network policy:

```bash
RUNNER=$(slicer vm fork "$COMMIT" --tag role=runner --quiet)
```

For a child with no egress, clear the inherited allow list and drop everything else:

```bash
JOB=arkade-run-$(cat /proc/sys/kernel/random/uuid)
RUNNER=$(slicer vm fork "$COMMIT" \
  --tag role=runner \
  --tag "job=$JOB" \
  --no-allow \
  --drop 0.0.0.0/0 \
  --quiet)

echo "$RUNNER"
```

`--drop 0.0.0.0/0` on its own is not enough when the host group has the default `--allow 0.0.0.0/0` rule. The allow rule takes precedence, so use `--no-allow` to clear it.

The DROP is enforced on the Slicer host, outside the guest. A root process inside the VM cannot remove it with `iptables`, ignore it by opening a raw socket, or bypass it by unsetting a proxy variable.

The fork command returns after `slicer-agent` has finalised the child's identity. If the client disconnects while waiting, the daemon continues the fork. Recover the allocated hostname through the unique job tag, then describe it:

```bash
RUNNER=$(slicer vm list --json | jq -r --arg tag "job=$JOB" \
  '.[] | select((.tags // []) | index($tag)) | .hostname' | head -n1)
slicer vm describe "$RUNNER"
```

## Use and delete the child

The child inherits the compiler, source tree, vendored dependencies, and Go build cache from the builder. Change one line and rebuild:

```bash
slicer vm exec "$RUNNER" -- \
  "cd /home/ubuntu/arkade && \
   sed -i \
     's/boot Linux microVMs instantly/boot Linux microVMs quickly/' \
     pkg/thanks.go && \
   /usr/local/go/bin/go build -mod=vendor -o ./arkade"
```

Delete the child when the job is complete:

```bash
slicer vm delete "$RUNNER"
```

The committed parent remains available for another fork.

## Reuse the builder step

A cache key lets an API-driven workflow skip the whole builder step on its next run. It is similar to a Docker build cache, but Slicer caches the complete committed disk rather than individual layers.

Look up the key before launching a builder:

```bash
KEY=arkade-go-v1
COMMIT=$(slicer vm commit list \
  --cache-key "$KEY" \
  --json | jq -r '.[0].commit_id // empty')

if [ -z "$COMMIT" ]; then
  BUILDER=$(slicer vm add runners \
    --persistent \
    --tag purpose=cold-fork-builder \
    --wait \
    --json | jq -r '.hostname')

  # Run the preparation commands from above.
  slicer vm shutdown "$BUILDER"
  COMMIT=$(slicer vm commit "$BUILDER" \
    --cache-key "$KEY" \
    --quiet)
fi

RUNNER=$(slicer vm fork "$COMMIT" --tag role=runner --quiet)
```

The caller owns the cache key. Include the inputs which make the builder result reusable, such as the toolchain version, dependency lock-file digest, or setup-script version. Change the key to invalidate the cached result.

The same lookup is available through `GET /vm/commits?cache_key=...`, `ListCommits()` in the [Go SDK](/platform/go-sdk/), and `client.commits.list()` in the [TypeScript SDK](/platform/typescript-sdk/).

## Organise committed parents

Add tags, labels, or a deterministic cache key when the calling application needs to find a committed parent later. Commit metadata is immutable, so set it on the initial `slicer vm commit` command, as shown above.

List all committed parents, or filter the list:

```bash
slicer vm commit list
slicer vm commit list --tag arkade
slicer vm commit list --cache-key arkade-go-v1
slicer vm commit list --source "$BUILDER"
```

Delete a committed parent when neither its source VM nor any forked child uses it:

```bash
slicer vm commit delete "$COMMIT"
```

## Cold fork, suspend, and restore

Cold fork and suspend solve different problems:

* `slicer vm commit` records disk state from a stopped VM. Each fork is a new VM which boots from that state.
* `slicer vm suspend` records memory, disk, and device state. `slicer vm restore` resumes the same VM later.

Persistent Firecracker VMs in Slicer, and VMs in Slicer for Mac, can suspend and restore today. A suspended VM cannot yet be forked into several children.

## Cold fork or a custom image?

A [custom Slicer image](/platform/custom-images/) is the better fit for a versioned base which needs to be distributed across several hosts. It can be built in CI, pushed to a registry, and referenced from a host group.

Cold fork is a leaner local loop. Prepare a running VM interactively, commit it, then fork locally without building a Dockerfile, pushing and pulling an OCI image, or unpacking it again on the Slicer host.

## See also

* [VM lifecycle](/platform/lifecycle/)
* [Networking](/reference/networking/)
* [Storage backends](/storage/overview/)
* [Build a custom image](/platform/custom-images/)
* [Go SDK cold forking example](https://github.com/slicervm/sdk/blob/master/examples/cold-fork/main.go)
