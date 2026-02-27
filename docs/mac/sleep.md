# Mac sleep behavior

Apple's sleep behaviour can get in the way of running local VMs, whatever solution you happen to use.

A fool-proof solution is to run `slicer vm suspend slicer-1` before closing the laptop lid or putting it into sleep. Then, later you can run `slicer vm restore slicer-1`. This relies on the suspend/resume functionality of Apple's Virtualization framework, and dumps all memory to disks. Slicer will sync the clock via its guest agent after restoring any suspended VM.

Running Slicer for Mac on a laptop means host sleep can affect VM lifecycle.

You can set a `sleep_action` inside each host group in `slicer-mac.yaml` (`slicer` and `sbox` blocks):

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
