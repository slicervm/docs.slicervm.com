# Run a task in a VM

This example shows how to run a one-shot task in a VM via API. You can also use the CLI if you wish.

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
#!/bin/bash
curl -i -X POST -d "$(cat /etc/hostname) booted\nUptime: $(uptime)" \
    https://webhook.site/f38eddbf-6285-4ff8-ae3e-f2e782c73d8f

sleep 1
sudo reboot
exit 0
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
