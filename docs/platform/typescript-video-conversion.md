# Example: Video Conversion (TypeScript)

A full end-to-end example that uses the [TypeScript SDK](/platform/typescript-sdk/) to create a microVM, install ffmpeg, transcode a video, and stream the binary result back — without an intermediate file on the guest disk.

The complete, runnable source is in [`examples/ffmpeg/`](https://github.com/slicervm/sdk-ts/tree/master/examples/ffmpeg) in the SDK repository.

For the Go equivalent, see [Video Conversion (Go)](/platform/video-conversion/).

## What this demonstrates

- `client.vms.create()` returning a `VM` handle that owns all per-VM operations.
- `vm.fs.writeFile()` — binary-safe file upload via native fs endpoints.
- `vm.execBuffered({ stdio: 'base64' })` — binary output streamed back as a `Buffer`, no second file copy.
- Server-side `wait: 'agent'` so the VM is ready the moment `create()` returns.
- Deterministic cleanup in a `finally` block.

## Set up Slicer

If you're on Linux, create a host group with no pre-allocated VMs:

```bash
slicer new sdk --count=0 --graceful-shutdown=false > sdk.yaml
sudo slicer up sdk.yaml
export SLICER_URL="http://127.0.0.1:8080"
export SLICER_TOKEN="$(sudo cat /var/lib/slicer/auth/token)"
```

On Slicer for Mac, the `sbox` host group is preconfigured:

```bash
export SLICER_URL=~/slicer-mac/slicer.sock
```

## Install the SDK

```bash
mkdir video-convert && cd video-convert
npm init -y
npm install @slicervm/sdk
npm install -D tsx typescript @types/node
```

## Get an input video

The example works with any video file. If you don't have one handy, generate a synthetic 3-second test pattern with ffmpeg — it's pure generated output, no copyright, and always reproducible:

```bash
ffmpeg -f lavfi -i testsrc=duration=3:size=640x480:rate=24 \
       -c:v libx264 -pix_fmt yuv420p input.mkv
```

For real footage use your own file, or a public-domain source like NASA's [Image and Video Library](https://images.nasa.gov/).

## The program

```ts
// convert.ts
import fs from 'node:fs/promises';
import path from 'node:path';
import { SlicerClient, GiB, type VM } from '@slicervm/sdk';

const HOST_GROUP = process.env.SLICER_HOST_GROUP ?? 'sbox';

async function main() {
  const [inputPath, outputPath] = process.argv.slice(2);
  if (!inputPath || !outputPath) {
    console.error('usage: convert.ts <input> <output>');
    process.exit(2);
  }

  const inputBytes = await fs.readFile(inputPath);
  console.log(`→ input ${path.basename(inputPath)} (${inputBytes.length} bytes)`);

  const client = SlicerClient.fromEnv();

  const vm: VM = await client.vms.create(
    HOST_GROUP,
    { cpus: 2, ramBytes: GiB(2), tags: ['ffmpeg-example'] },
    { wait: 'agent', waitTimeoutSec: 120 },
  );
  console.log(`→ VM ${vm.hostname} ready`);

  try {
    console.log('→ installing ffmpeg…');
    const install = await vm.execBuffered({
      command: '/bin/sh',
      args: ['-c',
        'apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -y -qq ffmpeg'],
      uid: 0,
      gid: 0,
    });
    if (install.exitCode !== 0) {
      throw new Error(`apt install failed: ${install.stderr}`);
    }

    console.log('→ uploading source…');
    await vm.fs.writeFile('/tmp/input', inputBytes);

    console.log('→ transcoding (H.264/AAC, 720p cap)…');
    const result = await vm.execBuffered({
      command: 'ffmpeg',
      args: [
        '-hide_banner', '-loglevel', 'error',
        '-i', '/tmp/input',
        '-vf', 'scale=-2:720',
        '-c:v', 'libx264', '-preset', 'medium', '-crf', '23',
        '-c:a', 'aac',
        '-movflags', '+frag_keyframe+empty_moov',
        '-f', 'mp4', 'pipe:1',
      ],
      stdio: 'base64', // ← binary-safe wire; result.stdout is a Buffer
    });

    if (result.exitCode !== 0) {
      throw new Error(`ffmpeg failed: ${result.stderr.toString('utf8')}`);
    }

    await fs.writeFile(outputPath, result.stdout);
    console.log(`→ wrote ${outputPath} (${result.stdout.length} bytes)`);
  } finally {
    await vm.delete().catch(() => {});
  }
}

main().catch((err) => {
  console.error('error:', err instanceof Error ? err.message : err);
  process.exit(1);
});
```

Run it:

```bash
npx tsx convert.ts input.mkv output.mp4
```

Expected:

```
→ input input.mkv (22636 bytes)
→ VM sbox-1 ready
→ installing ffmpeg…
→ uploading source…
→ transcoding (H.264/AAC, 720p cap)…
→ wrote output.mp4 (43121 bytes)
```

Verify the result with `ffprobe output.mp4` — it should report H.264 at 720p.

## Why `stdio: 'base64'`?

The default wire format for `exec` is UTF-8 text, which mangles arbitrary binary through JSON string escaping. `stdio: 'base64'` tells the daemon to base64-encode stdout/stderr frames; the SDK decodes them into `Buffer` values transparently. Without it you'd need to write the MP4 to a file in the VM and fetch it with a second `vm.fs.readFile` call — twice the disk I/O and a second round-trip.

## Speeding this up

Most of the wall time is `apt install ffmpeg`. For repeat runs:

* [Build a custom image](/platform/custom-images/) with ffmpeg pre-installed.
* Create the VM with `persistent: true` and relaunch it between jobs (`vm.shutdown()` → `vm.relaunch()`).

## See also

* [TypeScript SDK reference](/platform/typescript-sdk/) — all methods and types.
* [Video Conversion (Go)](/platform/video-conversion/) — same workflow, Go SDK.
* [REST API reference](/reference/api/) — the underlying HTTP endpoints.
* [Example source on GitHub](https://github.com/slicervm/sdk-ts/tree/master/examples/ffmpeg)
