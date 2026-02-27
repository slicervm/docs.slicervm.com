# Storage on Slicer for Mac

Slicer for Mac stores VM state using a base image flow with copy-on-write clones.

You can set storage size per host group with `storage_size` in `slicer-mac.yaml`.

Use `nG` style values (for example `15G`), and set at least `5G` to give enough initial headroom.

Sparse files are used, so disk space is not allocated up front. A declared size of `15G` represents logical size and grows as data is written.

## Image/bootstrap workflow

Each host group (`slicer`, `sbox`) follows the same process:

### First start

1. Pull the image from `config.image` and unpack it into the local OCI cache.
2. Build the host-group base image (for example `./slicer-base.img`).
3. Extract the kernel to `./kernel/<host_group>/Image`.
4. Create the first VM disk from that base image.

### New VM start (warm path)

1. Reuse the OCI cache.
2. Reuse the host-group base image.
3. Clone the VM disk with APFS copy-on-write:

```bash
cp -c ./slicer-base.img ./slicer-1.img
```

4. Start the VM from the clone.

If `cp -c` is unavailable, the tool falls back to a normal file copy.

## Why storage sizes look smaller than expected

`slicer-base.img`, `slicer-base`, and VM `.img` files are sparse.
A size like `15G` is a logical size and grows as data is written. Use the `nG` format and keep the value at least `5G`.

## Rebuild or reset local state

To force a full rebuild from OCI:

```bash
rm -f ./slicer-base.img
rm -rf ./kernel/slicer ./kernel/sbox
rm -rf ./oci-cache
```

To remove a single VM image and recreate it from the host-group base image:

```bash
rm -f ./slicer-1.img
```

## Troubleshooting

If your VM crashes for some reason, or the Mac's sleep mode causes an issue with the VM, you may find the `.img` file needs to be checked or repaired with the `e2fsck` utility.

```bash
e2fsck -f ./slicer-1.img
```

You can obtain `e2fsck` by installing the `e2fsprogs` package via `brew install e2fsprogs`. Brew is available separately at: [https://brew.sh/](https://brew.sh/).
