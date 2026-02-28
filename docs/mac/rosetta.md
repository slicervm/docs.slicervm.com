# Enable Rosetta for x86_64 binaries

Enable Rosetta if you need to run x86_64 Intel/AMD Linux binaries in your Slicer guest.
For more background on Rosetta, see Apple’s documentation: https://support.apple.com/en-us/102527

## Enable in `slicer-mac.yaml`

Edit the relevant host group in your `slicer-mac.yaml` and flip `rosetta` to `true`.

- `slicer` host group affects your persistent main VM (`slicer-1`).
- `sbox` host group affects sandbox VMs (`sbox-*`).

```diff
-  rosetta: false
+  rosetta: true
@@
-  rosetta: false
+  rosetta: true
```

## Install Rosetta in the guest VM (if wanted)

If your distribution needs Rosetta and `rosetta: true`, copy the helper into the guest and run it there.

```bash
slicer vm cp ./enable-rosetta.sh slicer-1:~/
```

```bash
slicer vm exec slicer-1 -- bash ~/enable-rosetta.sh
```

Keep this off unless you need x86_64 compatibility.
