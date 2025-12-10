# Cursor CLI Agent

Cursor's CLI can be used in a non-interactive way within a script or CI job. That makes it ideal for use as a [one-shot task](/tasks/userdata) with a SlicerVM, or a longer running VM that you connect to with a [remote VSCode IDE](/examples/remote-vscode.md).

Using this flow, you could create a powerful agent that uses your existing Cursor subscription, running with the "auto" model so it remains within the free tier.

Example use-cases:

* Web scraping, data extraction and analysis
* Code or content generation
* Analyze user-submitted content such as code, comments, or documents
* Image or video conversion
* Chat bots, and webhook receivers / responders - i.e. from GitHub, GitLab, Discord, Slack, etc.

Running within a microVM means you can run the agent in a secure, isolated environment, and you can easily discard the VM when done.

> Note: Overall, Cursor's CLI has some rough edges, and you may prefer to use [opencode](/examples/opencode-ai-agent), which at time of writing worked better headless and was more polished.
> 
> The CLI may require an initial interactive session to enable MCP usage and to "trust" the working directory.
> 
> The CLI can also hang indefinitely after responding to a prompt, even in --print mode.
> 
> The CLI seems to have no practical way to read a prompt from a file or a stdin pipe, so you have to use `$(cat prompt.txt)` which can end up reading subsequent commands into the prompt.

## Quick overview

* Create a [Cursor API key](https://cursor.com/dashboard?tab=background-agents) and save it for use with the CLI.
* Create a userdata script to pre-install the CLI, Playwright for accessing the web, and your Cursor API key.
* Boot up the VM, optionally, collect the results from the logs or from the disk via SCP or by mounting the disk image after shutdown.

## Userdata script

Populate the prompt section with your own inputs.

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

mkdir -p ~/.cursor/

echo 'export PATH="$HOME/.local/bin:$PATH"' >> ./.bashrc

# Add cursor CURSOR_API_KEY env variable
echo "export CURSOR_API_KEY=$(cat /run/slicer/secrets/cursor-api-key)" | tee -a ~/.bashrc

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
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--browser", "chromium",
        "--headless"
        ]
    }
  }
}

EOF

cat << EOF > ~/.cursor/cli-config.json
{
  "permissions": {
    "allow": [
      "*"
    ],
    "deny": []
  }
}

EOF

mkdir -p ~/.cursor/projects/home-ubuntu/

cat << EOF > ~/.cursor/projects/home-ubuntu/mcp-approvals.json
[
  "playwright-aced25c5e5b87b1c"
]

EOF

mkdir -p ~/.cache/ms-playwright/

cat << EOF > ~/prompt.txt
Visit the slicervm.com website and generate a JSON summary of the key elements, value-proposition,
and target audience. Use playwright-mcp, this has been pre-installed and is ready for your use.
EOF

source ~/.bashrc

# Belt and braces..
sudo chown -R "$USER:$USER" "$HOME"

cursor-agent mcp list
cursor-agent mcp list-tools playwright

time cursor-agent --model auto --force --output-format=text --print "$(cat ./prompt.txt)" > ~/cursor.log
```

Note: at time of writing, a `mcp-approvals.json` file had to be created to get the MCP tool calls to work. Hopefully the cursor team will negate this requirement for headless use in a future release.

The approval file is not required if you do not use MCP tools.

## Config file

Create a slicer VM secret for your Cursor API key (Obtained from: https://cursor.com/dashboard?tab=background-agents)

Run the following, then paste in your API key, hit enter once, then Control + D to save the file.

```bash
sudo mkdir .secrets
# Ensure only root can read/write to the secrets folder.
sudo chmod 700 .secrets

sudo cat > .secrets/cursor-api-key
```

Use `slicer new` to generate a configuration file:

```bash
slicer new cursor \
  --userdata-file cursor.sh \
  > cursor.yaml
```
Save the resulting file as `cursor.yaml`.

## Give it a test run

Now run:

```bash
sudo slicer up ./cursor.yaml
```

## Taking it further

You can learn more about the [Cursor CLI here](https://docs.cursor.com/en/cli/overview) and its [headless usage here](https://docs.cursor.com/en/cli/headless)

If your agent doesn't need to browse the web, then you can speed up the boot by removing the Playwright instructions.

If you have additional MCP servers, you can add them into the userdata and ~/.cursor/mcp.json file.

To make the system boot up even quicker, you can derive your own custom image with the CLI and playwright pre-installed. Then the userdata will only be used to set up the API key, and to provide the prompt.

Finally, if you are automating access, you could automate the workflow by [creating a microVM via the REST API](/examples/run-a-task), and using `scp` to collect the generated code or results from the agent.
