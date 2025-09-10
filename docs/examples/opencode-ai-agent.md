# Run an AI Agent with OpenCode

This example is very minimal and can be tuned and adapted in many ways to suit your workflow.

The general idea is that everything is orchestrated via an initial userdata script:

* Install opencode CLI.
* Configure model authentication using a pre-defined config file `~/.local/share/opencode/auth.json` from another machine like your workstation.
* Set the agent working away as a systemd service, with a prompt set in the userdata file.

This example is one-shot, so it's designed to run to completion once, without any further interaction.

## Example config

The slicer config will be adapted from the [walkthrough](/getting-started/walkthrough). When you create the YAML, name it `opencode.yaml`.

Now, just add the `userdata_file` in the hostgroup section.

`userdata_file: ./opencode.sh`

And customise the `ssh_keys` or `github_user` fields so you can connect via SSH to review the logs, and/or scp to pull out any generated code.

On a computer where you've pre-installed and authenticated opencode, create a base64 representation of the auth file:

```bash
# MacOS
base64 --wrap 9999 ~/.local/share/opencode/auth.json
# Linux
base64 --cols 9999 ~/.local/share/opencode/auth.json
```

Then, create opencode.sh:

```bash
#!/usr/bin/env bash
# Ubuntu only. Requires: curl + arkade preinstalled.
# Installs opencode, sets pre-auth, creates a daemonized systemd unit "opencode.service".

set -euo pipefail

# === set this ===
# set base64 of ~/.local/share/opencode/auth.json}
export OPENCODE_AUTH_JSON_B64=""
# =================

# Install opencode -> /usr/local/bin
arkade get opencode --path /usr/local/bin >/dev/null
chown ubuntu /usr/local/bin/opencode
chmod +x /usr/local/bin/opencode

# Prep dirs & auth for user "ubuntu" (no group assumption)
for d in /home/ubuntu/workdir /home/ubuntu/.local/share/opencode /home/ubuntu/.local/state /home/ubuntu/.cache; do
  mkdir -p "$d"
  chown ubuntu "$d"
done
echo "$OPENCODE_AUTH_JSON_B64" | base64 -d > /home/ubuntu/.local/share/opencode/auth.json
chown ubuntu /home/ubuntu/.local/share/opencode/auth.json
chmod 600 /home/ubuntu/.local/share/opencode/auth.json

# Task payload (edit as needed)
cat >/home/ubuntu/workdir/task.txt <<'EOF'
Create a new Go program with an HTTP server.
Add GET /healthz returning 201 "Created".
Print a git diff at the end.

Use "arkade system install golang" to install Go into the environment. Then test the program with "go run main.go" and curl localhost:8080/healthz - make sure you kill the program after confirming the correct HTTP code was returned.

Set the PATH variable to include /usr/local/go/bin so that the go command is found.
EOF

chown ubuntu /home/ubuntu/workdir/task.txt
chmod 600 /home/ubuntu/workdir/task.txt

# systemd service (daemonized under systemd)
cat >/etc/systemd/system/opencode.service <<'EOF'
[Unit]
Description=OpenCode one-shot (daemonized under systemd)
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/workdir
Environment=HOME=/home/ubuntu
Environment=XDG_STATE_HOME=/home/ubuntu/.local/state
Environment=XDG_CACHE_HOME=/home/ubuntu/.cache
Environment=XDG_DATA_HOME=/home/ubuntu/.local/share
ExecStart=/usr/bin/env bash -lc '/usr/local/bin/opencode run "$(cat /home/ubuntu/workdir/task.txt)" -m opencode/grok-code'
Restart=no
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now opencode.service
```

Just before you start up the VM, make sure you customise the prompt to have the agent do whatever it is you need.

```bash
cat >/home/ubuntu/workdir/task.txt <<'EOF'

Prompt goes here
Over multiple lines

EOF
```

Start up the VM i.e. `sudo -E slicer up ./opencode.yaml`

**What if you want to copy in a private Git repository?**

One option is to include an SSH key for the agent/ubuntu user, so that it can clone the repository directly from GitHub or another Git server. To keep permissions tight, you could also simply `scp` the code into the microVM after it has booted, like below.

If you want to work on a private Git repository, simply have the systemd unit wait until it finds a folder within the workdir folder, and then scp the code from your host after the microVM has booted.

So if the directory was named i.e. `arkade`, and we'd cloned it locally, you could amend the `opencode` systemd unit like so:

```bash
[Service]
ExecStartPre=/usr/bin/env bash -c 'while [ ! -d /home/ubuntu/workdir/arkade ]; do sleep 5; done'
ExecStart=/usr/bin/env bash -lc '/usr/local/bin/opencode run "$(cat /home/ubuntu/workdir/task.txt)" -m opencode/grok-code'
```


## View the results

Once the VM is running, you can check the status of the `opencode` service.

The code will be written to the `$HOME/workdir` directory.

```bash
ssh ubuntu@192.168.137.2

sudo journalctl -u opencode.service -f

cd workdir
find .

git diff
```

In the example of the preloaded prompt from above, we saw in `~/workdir/main.go`:

```bash
package main

import (
        "fmt"
        "net/http"
)

func main() {
        http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
                w.WriteHeader(http.StatusOK)
                fmt.Fprint(w, "ok")
        })
        http.ListenAndServe(":8080", nil)
}
```

To copy out the workdir, run something like this on your host:

```bash
mkdir -p agent-runs
cd agent-runs
scp -r ubuntu@192.168.137.2:/home/ubuntu/workdir .
```

## Further thoughts

This was a very basic example to get you thinking - you could write your own program and use that to drive the whole interaction, rather than using systemd and bash.

SSH could also be a better way to interact with the agent, rather than passing an initial prompt and token via the userdata file.

