# Remote VSCode

Let's say that you work away from your computer often, or maybe just want to have a remote IDE that you can access from any kind of device like an iPad, or a Chromebook.

You can run Microsoft's VSCode server in a VM using Slicer and connect to it remotely. Microsoft also offer free tunnels to connect to your server with authentication enabled via GitHub or Microsoft accounts.

It's incredibly convenient.

Alex recently migrated his blog from Ghost 1.x which had a built-in markdown editor, over to a static Next.js site. This example lets him edit the content of his blog without having a laptop with him. This whole setup could be run on a Raspberry Pi 5, or a cheap N100 mini computer.

![VSCode tunneled to the web with built-in authentication and Copilot helping out](/images/vscode-blog.jpg)
> VSCode tunneled to the web with built-in authentication and Copilot helping out

## Set up your initial VM

Use the [walkthrough](/getting-started/walkthrough) to get a basic VM with the specs you want, with your SSH keys pre-installed, and use a disk image as backing to make it persistent.

When you create your config, you can [add userdata](/tasks/userdata) to install the VSCode server, so it's already there by the time you get in.

The below is based upon the [Official VSCode Documentation](https://code.visualstudio.com/docs/setup/linux).

```yaml
config:
  host_groups:
  - name: vscode
    userdata: |
        #!/bin/bash

        (
        sudo apt-get install -qy curl gpg apt-transport-https

        curl -sLS -o - https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
        sudo install -D -o root -g root -m 644 microsoft.gpg /usr/share/keyrings/microsoft.gpg

        cat > /etc/apt/sources.list.d/vscode.sources << EOF
        Types: deb
        URIs: https://packages.microsoft.com/repos/code
        Suites: stable
        Components: main
        Architectures: amd64,arm64,armhf
        Signed-By: /usr/share/keyrings/microsoft.gpg
        EOF

        sudo apt install -qy apt-transport-https
        sudo apt update
        sudo apt install -qy code --no-install-recommends
        )
```

You can watch the installation by adding the [fstail](https://github.com/alexellis/fstail) tool via `arkade get fstail`.

Then run the following to attach to any VM log files that are detected:

```bash
sudo fstail /var/log/slicer
```

It seemed to take about 16s to get all of the above installed and configured.

## Start the VSCode server

Log in via ssh i.e. `ssh ubuntu@192.168.137.2`

To use Microsoft's built-in tunneling service, you need to accept their license terms:

```bash
code tunnel --accept-server-license-terms \
    --name $(hostname)
```

You'll be presented with a URL to authenticate with GitHub using a device code.

Then you'll get a URL for your remote VSCode server.

### Alternative without Microsoft's tunneling service

Alternatively, you can run the server without tunneling, and connect directly to it over HTTPS.

```bash
code serve-web
```

A link will be printed out pointing to `http://localhost:8080` including a token used to authenticate.

Then port-forward the port 8000 to your local machine:

```bash
ssh -L 8000:127.0.0.1:8000 ubuntu@192.168.137.2
```

Or launch an [inlets HTTPS tunnel via inletsctl](https://docs.inlets.dev/tutorial/automated-http-server/) to get a public URL with TLS enabled. You can enable auth using basic auth or OAuth2 [with the inlets command](https://docs.inlets.dev/tutorial/http-authentication/).

## Customise the environment

Install any tooling you want such as Node, or Go, Python, Docker etc.

If you use Copilot, log into your Microsoft account, and then you can enable the Copilot extension in your remote VSCode server.

## Another approach: VScode with SSH

Instead of launching a browser-based VSCode server, you can use your local VSCode installation to connect to the microVM over SSH.

* Any agents you run can work in YOLO mode, without risking your content, confidential data, or any risk of breaking your main host.
* You can create, delete, and iterate on microVMs easily, and if you break something, just delete it and start again.
* You can get access to a full Linux host, whilst working on MacOS or Windows.
* You can test programs out on an Arm host, even if your main host is x86_64.

You can also use VSCode's built-in SSH support to connect to your VM. This allows you to use your local VSCode installation to edit files on the remote machine, with all your familiar settings already available.

1. Install the [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) extension in your local VSCode.
2. Configure your SSH settings to connect to your VM.
3. Open a remote window in VSCode and start editing files on your VM.

This approach gives you the full power of your local VSCode environment whilst making sure any packages you install, or changes you make, do not affect your main host.