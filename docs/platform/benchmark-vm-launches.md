# Benchmark VM Launches

`slicer bench` benchmarks VM launch latency on a Slicer host. It runs repeated
tests with varying numbers of concurrent VM launches and reports P50, P95,
maximum latency, and failures.

Platforms often create sandboxes in bursts, so launch latency affects how soon
users and workloads can start. Use the benchmark to validate a host
configuration and establish expected launch times under load.

Run the benchmark against either:

* A [dedicated Slicer instance for benchmarking](#set-up-a-benchmark-instance)
* An [existing Slicer instance](#benchmark-an-existing-instance)

## Set up a benchmark instance

`slicer new --bench` generates a Slicer configuration optimized for minimal
boot latency during benchmarking.

Create a dedicated directory so that the
configuration and API socket are easy to find:

```bash
mkdir -p ~/slicer-bench
cd ~/slicer-bench

slicer new --bench > bench.yaml
```

The preset creates a `bch` host group with:

* the [minimal Slicer image](/reference/images/#minimal-image)
* devmapper storage and non-persistent disks
* isolated networking
* no VMs created at startup (`count: 0`)
* 1 vCPU and 1 GiB RAM per VM
* no SSH key discovery or graceful shutdown
* a Unix socket at `./slicer.sock`

Devmapper must already be configured on the host. See [devmapper
storage](/storage/devmapper/) for installation instructions.

Explicit flags override the preset. For example, use image storage when
devmapper is unavailable:

```bash
slicer new --bench --storage image > bench.yaml
```

Start Slicer with the generated configuration:

```bash
sudo -E slicer up bench.yaml
```
Open another terminal in the same directory and run:

```bash
cd ~/slicer-bench
slicer bench \
  --concurrency 1,5,10 \
  --runs 3
```

This tests three concurrency levels: 1, 5, and 10 simultaneous VM launches.
Each level is repeated three times, for nine independent runs in total.

Each run starts clean and launches all VMs at approximately the same time.
Slicer records how long each VM takes to meet the selected completion condition,
then deletes the VMs before the next run.

## Benchmark an existing instance

Run `slicer bench` against any existing Slicer instance:

```bash
slicer bench \
  --url ./slicer.sock \
  --concurrency 1,2,4,8 \
  --runs 3
```

This tests concurrent launches of 1, 2, 4, and 8 VMs, repeating each test three times.

By default, `slicer bench` uses the first available host group returned by the
server. Use `--hostgroup` to select a specific host group instead:

```bash
slicer bench \
  --url ./slicer.sock \
  --hostgroup sandbox \
  --concurrency 1,2,4,8 \
  --runs 3
```

Use the standard API connection flags to select the Slicer instance and
authenticate: `--url`, `--socket`, `--token`, and `--token-file`.

## Choose the completion condition

`--wait` determines when each launch is complete:

| Value | Measurement |
| --- | --- |
| `agent` | From issuing the VM-create operation until the guest agent reports ready. This is the default. |
| `userdata` | From issuing the VM-create operation until userdata completes. Use this when the host group has non-empty userdata. |
| `none` | From issuing the VM-create operation until the create request returns, without waiting for guest readiness. |

For example, to include userdata execution in the launch latency:

```bash
slicer bench \
  --hostgroup sandbox \
  --concurrency 1,5,10 \
  --runs 3 \
  --wait userdata
```

## Results and output formats

The default text output summarizes all successful VM observations at each
concurrency level. It reports P50, P95, the maximum successful latency, and
failures as a fraction of total launches:

```text
CONCURRENCY  P50    P95    MAX    FAILURES
1            1.21s  1.34s  1.34s  0/3
5            1.47s  1.82s  1.82s  0/15
10           1.91s  2.63s  2.63s  0/30
```

If any launches fail, `slicer bench` returns a non-zero exit status.

Use `--verbose` to add every VM observation, including its concurrency level,
run number, VM name, duration, and status:

```bash
slicer bench \
  --hostgroup bch \
  --concurrency 1,5,10 \
  --runs 3 \
  --verbose
```

Use `--json` to include the raw observations and summaries in structured
output:

```bash
slicer bench \
  --hostgroup bch \
  --concurrency 1,5,10 \
  --runs 3 \
  --json > results.json
```

## Keep VMs for debugging

By default, the benchmark deletes every run's VMs before starting the next run.
Use `--keep` only when you need to inspect them after a failure:

```bash
slicer bench \
  --hostgroup bch \
  --concurrency 5 \
  --runs 1 \
  --keep
```

Kept VMs consume host resources and can affect later measurements. Delete them
after debugging before running another comparison.
