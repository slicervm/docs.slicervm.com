# Userdata for Slicer VMs

Userdata can be used to customise VMs on first boot. Another option is to [build a custom image](/tasks/custom-image).

This can include anything from installing packages, configuring services, or setting up user accounts.

If you wanted to run AI agents within Slicer, you could use userdata to install a custom AI framework, library, or agent for Remote Procedure Control (RPC).

## Userdata in the config file

```diff
config:
  host_groups:
  - name: agent
+   userdata: |
+     # Install Ollama
+     curl -fsSL https://ollama.com/install.sh | sh
+
+     # Install a hard-coded SSH key for remote administration
+
+     echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3... user@host" | tee -a /home/ubuntu/.ssh/authorized_keys
+     chmod 600 /home/ubuntu/.ssh/authorized_keys
+     chown ubuntu:ubuntu /home/ubuntu/.ssh/authorized_keys
```

You could also install something like Docker like this:

```diff
config:
  host_groups:
  - name: agent
+   userdata: |
+     # Install Ollama
+     curl -fsSL https://get.docker.com | sh
```

## Userdata via the CLI

You can add and remove VMs via Slicer's API directly using HTTP, or using the Slicer CLI itself as a client.

If you had a hostgroup named `agents`, you could run:

```bash

USERDATA=$(echo '# Install Ollama\ncurl -fsSL https://ollama.com/install.sh | sh' | jq -Rs .)

curl -sLSf http://127.0.0.1:8080/hostgroup/agents/nodes \
    -H "Content-Type: application/json" \
    -d "{
      \"userdata\": $USERDATA
    }"
```

For longer userdata scripts, you can save them to a file and reference it:

```bash
# Create userdata.sh
cat > userdata.sh << EOF
#!/bin/bash

# Install Docker
curl -fsSL https://get.docker.com | sh

# Add user to docker group
usermod -aG docker ubuntu
EOF

# Use the file in your API call
curl -sLSf http://127.0.0.1:8080/hostgroup/agents/nodes \
    -H "Content-Type: application/json" \
    -d "{
      \"userdata\": $(cat userdata.sh | jq -Rs .)
    }"
```

Or you can use the CLI for more convenience:

```bash
sudo -E slicer vm add \
    --userdata ./userdata.sh
```

