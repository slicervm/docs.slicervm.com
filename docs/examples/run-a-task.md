# Run a task in a VM

This example shows how to run a one-shot task in a VM via API. You can also use the CLI if you wish.

Tasks could include: CI jobs, databases for testing, isolated environments for privileged AI agents, crons, batch jobs and other kinds of temporary processes.

<iframe width="560" height="315" src="https://www.youtube.com/embed/5RjtVM4bvp0?si=2TPpSKn9YXFw_Nnt" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> Watch a demo of the tutorial to see how fast it is to launch microVMs for one-shot tasks.

## Tutorial

Create an empty hostgroup configuration.

For the fastest possible boot times, [use ZFS for storage](/storage/zfs).

If you don't have ZFS set up yet, you can simply replace the storage option with something like:

```yaml
storage: image
storage_size: 20G
```

Create `tasks.yaml`:

```yaml
config:
  host_groups:
  - name: task
    storage: zfs
    count: 0
    vcpu: 1
    ram_gb: 2
    network:
      bridge: brtsk0
      tap_prefix: tsktap
      gateway: 192.168.138.1/24

  github_user: alexellis

  api:
    bind_address: 127.0.0.1
    port: 8081

  kernel_image: "ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-latest"
  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker
```

Now start up slicer:

```bash
sudo -E slicer up ./tasks.yaml
```

Now set up a HTTP endpoint using a free service like [ReqBin.com](https://reqbin.com/post-online) or [webhook.site](https://webhook.site).

Write a userdata script to send a POST request to your HTTP endpoint on boot-up, then have it exit.

Save task.sh:

```bash
cat > task.sh <<'EOF'
#!/bin/bash
curl -i -X POST -d "$(cat /etc/hostname) booted\nUptime: $(uptime)" \
    https://webhook.site/f38eddbf-6285-4ff8-ae3e-f2e782c73d8f

sleep 1
sudo reboot
exit 0
EOF
```

Then run your task by booting a VM with the script as its userdata:

```bash
curl -isLSf http://127.0.0.1:8081/hostgroup/task/nodes \
    -H "Content-Type: application/json" \
    --data-binary "{
  \"userdata\": $(cat ./task.sh | jq -Rs .)
}"
```

Check your HTTP bin for the results.

You can also run this in a for loop:

```bash
for i in {1..5}
do
  curl -sLSf http://127.0.0.1:8081/hostgroup/task/nodes \
      -H "Content-Type: application/json" \
      --data-binary "{\"userdata\": $(cat ./task.sh | jq -Rs .)}"
done
```

![Each of the 5 tasks that executed and exited, posted to the endpoint](/images/tasks-web.png)
> Each of the 5 tasks that executed and exited, posted to the endpoint

## Optimise the image for start-up speed

After various Kernel modules are loaded, and the system has performed its self-checking, your code should be running at about the 2.5s mark, or a bit earlier depending on your machine.

To optimise the boot time further for one-shot use-cases, the SSH host key regenerate step that is present on start-up. It can add a few seconds to the boot time, especially if entropy is low on your system.

You can derive your own image to use, with this disabled:

```
FROM ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest

RUN systemctl disable regen-ssh-host-keys &&
    systemctl disable ssh && \
    systemctl disable sshd && \
    systemctl disable slicer-vmmeter
```

After SSH is disabled, the only way to debug a machine is via the Slicer agent using `slicer vm exec` to get a shell.

You can also disable `slicer-ssh-agent` (not actually a full SSH daemon), however the `slicer vm` commands will no longer work.

If you publish an image to the Docker Hub, make sure you include its prefix i.e. `docker.io/owner/repo:tag`.

### Cache the Kernel to a local file

Rather than downloading an extracting the Kernel on each run of Slicer, you can extract a given vmlinux file and tell the YAML file to use that.

The Kernel must agree with the root filesystem image, which means using a proper tag and not a `latest` tag.

Why? The Kernel is built as a vmlinux, however its modules are packaged into the root filesystem image.

Run the following:

```bash
$ arkade get crane
$ crane ls ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-latest

5.10.240-x86_64-3d7a67d1683b524b4128ad338f90b1da710f2fd9
5.10.240-kvm-x86_64-3d7a67d1683b524b4128ad338f90b1da710f2fd9
5.10.240-x86_64-ea04b63b9117c966a57e17e1bc1bfcf713cd6276
5.10.240-x86_64-bb71bdd1cd06bad2cc11f8ab3f323c3f19d41c8b
5.10.240-x86_64-2f2ebc0bbefe128aa3061e6ea6806cbcdc975208
6.1.90-aarch64-2f2ebc0bbefe128aa3061e6ea6806cbcdc975208
6.1.90-aarch64-5c59e9b9b08eea49499be8449099291c93469b80
5.10.240-x86_64-5c59e9b9b08eea49499be8449099291c93469b80
```

Pick a stable for your architecture i.e. `x86_64-SHA` or `aarch64-SHA`, then run:

```bash
$ arkade oci install ghcr.io/openfaasltd/actuated-kernel:5.10.240-x86_64-2f2ebc0bbefe128aa3061e6ea6806cbcdc975208 --output ./
$ ls
vmlinux
```

Next, find the matching root filesystem image:

```bash
$ crane ls ghcr.io/openfaasltd/slicer-systemd
```

Use it in your YAML file, replacing `kernel_image` with `kernel_file`:

```yaml
  kernel_file: "./vmlinux"
  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-2f2ebc0bbefe128aa3061e6ea6806cbcdc975208"
```

