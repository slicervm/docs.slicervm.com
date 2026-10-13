# Run a Task in Slicer

If your product needs to run background jobs, cron-style tasks, or headless AI agents in isolated environments, you can launch microVMs on demand and tear them down when the work is done. The `slicer` CLI is the quickest way to try it, and the same calls are available through the REST API and SDKs.

Use-cases include:

* Background jobs and batch processing as part of a SaaS product
* Running headless AI coding agents in isolated microVMs
* Cron jobs, scheduled tasks, and serverless-style functions
* CI builds or compiling untrusted customer code
* On-demand IDEs for pull request development or review

There is no limit on how long a task VM runs. It could be 250ms to process a webhook, 48 hours for fine-tuning, or several weeks. Launch task VMs with `--rm` so their disk is discarded as soon as the task exits.

<iframe width="560" height="315" src="https://www.youtube.com/embed/5RjtVM4bvp0?si=2TPpSKn9YXFw_Nnt" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

> Watch a demo of the tutorial to see how fast it is to launch microVMs for one-shot tasks.

## Tutorial

Each task in this tutorial is a short script. It boots, posts a result back to an HTTP endpoint on the host, then exits. Slicer removes the VM as soon as it does.

### Create a host group for tasks

```bash
mkdir -p ~/slicer-tasks && cd ~/slicer-tasks

slicer new task \
  --min \
  --net isolated \
  --count 0 \
  --api-port 8081 \
  > tasks.yaml
```

* `--min` uses a slimmer image that boots faster. It leaves out the Kernel options needed for Docker and Kubernetes, which one-shot tasks rarely need.
* `--net isolated` gives each task its own network namespace. A task can reach the internet, but not other VMs. Use `--allow` and `--drop` to restrict where it can connect to.
* `--count 0` starts with no VMs, so each task is launched on demand. Without it, `slicer new` defaults to one VM, which would start as soon as Slicer does.
* `--api-port 8081` keeps this daemon's API separate from any other Slicer instance on the host, which uses port 8080 by default.

For the fastest launches, add `--storage devmapper` to clone each task's disk from a snapshot. Install Slicer with `--devmapper` to enable it, which works with a loopback file if you don't have a spare disk. [ZFS](/storage/zfs) is just as fast. With the default `image` storage, each task takes a second or two longer to launch.

Start Slicer:

```bash
sudo slicer up ./tasks.yaml
```

### Collect the results

Tasks on an isolated network can't reach the host through their gateway, but they can reach it on its own IP address, just as other machines on your network do. Find it with `hostname -I`.

In a new terminal, save this small HTTP server as `receiver.py`. It prints whatever the tasks send back:

```python
import http.server

class Handler(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        body = self.rfile.read(int(self.headers["Content-Length"]))
        print(body.decode().strip(), flush=True)
        self.send_response(200)
        self.end_headers()

    def log_message(self, *args):
        pass

http.server.HTTPServer(("0.0.0.0", 9000), Handler).serve_forever()
```

Run it:

```bash
python3 receiver.py
```

It listens on every interface, so stop it with `Control+C` once you're done. In a real product, the task would write to your own API, a queue, or object storage instead.

### Write the task

The task is a userdata script that runs once the VM has booted. Save it as `task.sh`, replacing `192.168.1.34` with your host's IP:

```bash
#!/bin/bash
curl -sS -X POST --data-binary "$(hostname) ran its task. $(uptime)" \
  http://192.168.1.34:9000/

sudo reboot
```

The script ends with `reboot` rather than `poweroff`. Firecracker exits when its guest reboots, and Slicer then cleans up the VM. A `poweroff` only halts the guest, which leaves the VM running.

### Run the task

Point the CLI at the task daemon's API, then launch a task:

```bash
export SLICER_URL=http://127.0.0.1:8081

slicer vm add task --rm --userdata-file ./task.sh
```

```
VM created
  Hostname: task-1
  Group:    task
  Arch:     amd64
  IP:       172.16.0.2
  Specs:    2 vCPU, 4GiB RAM, GPUs: 0
  Persistent: false
  Duration: 1.698s
```

`--rm` makes the VM ephemeral, so its disk is discarded when the task exits.

Now run a batch of five:

```bash
for i in {1..5}; do
  slicer vm add task --rm --userdata-file ./task.sh
done
```

The receiver prints a line as each task finishes, including the first one you launched:

```
task-1 ran its task.  09:03:24 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
task-2 ran its task.  09:03:29 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
task-3 ran its task.  09:03:31 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
task-4 ran its task.  09:03:33 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
task-5 ran its task.  09:03:34 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
task-6 ran its task.  09:03:36 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
```

With the default `image` storage, each task took under two seconds from launch to reporting back. With `devmapper`, `slicer vm add` returns in a few milliseconds. In a separate run, a batch of nine tasks finished within three seconds.

Once the tasks have exited, nothing is left behind:

```bash
slicer ls
```

```
No VMs found
```

When using Cloud Hypervisor for GPU support, pass `--gpus` to `slicer vm add` to allocate GPUs from the host into the task's VM. This is useful for batch inference, or AI jobs that benefit from direct access to a local LLM.

### Breakglass: launch a task with curl

The `slicer` CLI is a client for Slicer's REST API. When you're integrating Slicer into your own product, or the CLI isn't to hand, you can call the API directly:

```bash
curl -sSf http://127.0.0.1:8081/hostgroup/task/nodes \
  -H "Authorization: Bearer $(cat /var/lib/slicer/auth/token)" \
  -H "Content-Type: application/json" \
  --data-binary "{
    \"persistent\": false,
    \"userdata\": $(jq -Rs . < ./task.sh)
  }"
```

`jq` turns the script into a JSON string. `"persistent": false` is the API's equivalent of `--rm`. See the [REST API reference](/reference/api) for every field, or use the [Go SDK](/platform/go-sdk) or [TypeScript SDK](/platform/typescript-sdk) from your own code.
