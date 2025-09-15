# Cursor CLI Agent

Cursor's CLI can be used in a non-interactive way within a script or CI job. That makes it ideal for use as a [one-shot task](/tasks/userdata) with a SlicerVM, or a longer running VM that you connect to with a [remote VSCode IDE](/examples/remote-vscode.md).

Using this flow, you could create a powerful agent that uses your existing Cursor subscription, running with the "auto" model so it remains within the free tier.

Example use-cases:

* Web scraping, data extraction and analysis
* Code or content generation
* Image or video conversion
* Chat bots, and webhook receivers / responders - i.e. from GitHub, GitLab, Discord, Slack, etc.

Running within a microVM means you can run the agent in a secure, isolated environment, and you can easily discard the VM when done.

## Quick overview

* Create a [Cursor API key](https://cursor.com/dashboard?tab=background-agents) and save it for use with the CLI.
* Create a userdata script to pre-install the CLI, Playwright for accessing the web, and your Cursor API key.
* Boot up the VM, optionally, collect the results from the logs or from the disk via SCP or by mounting the disk image after shutdown.

## Userdata script

Populate the prompt section, and the API key with your own inputs.

Save as cursor.sh:

```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET_USER=ubuntu

# If we're root, re-exec this script as $TARGET_USER (login shell so HOME/env are correct)
if [[ $EUID -eq 0 ]]; then
  if ! id -u "$TARGET_USER" >/dev/null 2>&1; then
    echo "User '$TARGET_USER' does not exist" >&2
    exit 1
  fi

  # Option A: run the file path directly (keeps $0 correct; requires the user can read the file)
  # exec sudo -u "$TARGET_USER" -i bash "$0" "$@"

  # Option B: stream the script via stdin (works even if the file isn't readable by $TARGET_USER)
  exec sudo -u "$TARGET_USER" -i bash -s -- "$@" < "$0"
fi

# From here on, we are the ubuntu user
echo "Running as $(whoami), HOME=$HOME"
# cd to HOME
cd

curl https://cursor.com/install -fsS | sudo -E bash

mkdir -p .cursor/

echo 'export PATH="$HOME/.local/bin:$PATH"' >> ./.bashrc

# Set your CURSOR_API_KEY below - obtained from: https://cursor.com/dashboard?tab=background-agents
echo "export CURSOR_API_KEY=" | tee -a ~/.bashrc

# Install playwright mcp tool for browsing web / fetching content
# These steps could be built into a custom Slicer image to speed up boot time.
sudo -E arkade system install node
npm i -D @playwright/test
npm i -D @playwright/mcp@latest

npx playwright install --with-deps chromium
#npx playwright install --with-deps webkit

cat << EOF > ~/.cursor/mcp.json
{
    "mcpServers": {
        "playwright-mcp": {
            "command": "npx",
            "args": [
                "@playwright/mcp@latest"
            ]
        }
    }
}
EOF

mkdir -p ~/.cache/ms-playwright/

cat << EOF > ~/prompt.txt
Visit the slicervm.com website and generate a JSON summary of the key elements, value-proposition,
and target audience. Use playwright-mcp, this has been pre-installed and is ready for your use.
EOF

source ~/.bashrc

# Belt and braces..
sudo chown -R ubuntu *

# Present your task/prompt in non-interactive mode for the agent/CLI:
# Output format also supports JSON
time cursor-agent --output-format=text --print <<< $(cat ~/prompt.txt) > ~/cursor.log

echo "Done" > cursor-done.txt
date >> ~/cursor.log
```

## Config file

Save the config file from the [walkthrough](/getting-started/walkthrough).

Then add the userdata field within the hostgroup:

```diff
config:
  hostgroups:
    - name: vm
+     userdata: ./cursor.sh
```

Save the resulting file as `cursor.yaml`.

## Give it a test run

Now run:

```bash
sudo slicer up ./cursor.yaml
```

## Taking it further

If your agent doesn't need to browse the web, then you can speed up the boot by removing the Playwright instructions.

If you have additional MCP servers, you can add them into the userdata and ~/.cursor/mcp.json file.

To make the system boot up even quicker, you can derive your own custom image with the CLI and playwright pre-installed. Then the userdata will only be used to set up the API key, and to provide the prompt.

Finally, if you are automating access, you could automate the workflow by [creating a microVM via the REST API](/examples/run-a-task), and using `scp` to collect the generated code or results from the agent.
