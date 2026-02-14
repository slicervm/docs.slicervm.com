# Mac sleep behavior

Running Slicer for Mac on a laptop means host sleep can affect VM lifecycle. Set `sleep_action` inside each host group in `slicer-mac.yaml` (`slicer` and `sbox` blocks):

```yaml
  - name: slicer
    count: 1
    share_home: "~/"
    sleep_action: prevent
  - name: sbox
    count: 0
    sleep_action: prevent
```

On a Mac, sleep can start when:

- the screen goes dark after idle time,
- you close the laptop lid,
- you choose **Sleep** from the Apple menu,
- scheduled wake/sleep policies or power settings trigger it.

## `sleep_action` options

- `none`: no explicit action.
- `shutdown`: request VM shutdown on host sleep.
- `prevent`: keep macOS awake while VMs are running.
- `suspend`: snapshot VM state and restore on wake.

## What happens

### Sleep event

- `shutdown`: request clean VM shutdown.
- `suspend`: write snapshot state and stop the VM (fallback to shutdown if snapshot is unavailable).
- `none` / `prevent`: no shutdown or snapshot action.

### Wake event

- `suspend`: on wake, wait 5 seconds for the system to settle, then run state-aware restore.
- `none`, `shutdown`, `prevent`: no restore action.
