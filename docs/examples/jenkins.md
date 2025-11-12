## Jenkins

[Jenkins](https://www.jenkins.io/) is a popular CI/CD tool with a rich history in the industry. Compared to newer products like GitHub Actions, it has much less complexity, and when used with Slicer, jobs launch almost instantly.

There are two use-cases for Jenkins with Slicer:

1. Run a Jenkins master and add build-slaves separately or via a public cloud plugin (EC2, GCE, etc)
2. Add ephemeral build slaves to a Jenkins master (existing, or hosted in Slicer)

## Learn more

Read how Slicer works and compares to the Docker( Docker In Docker / Docker Socket), Kubernetes (privileged Pod), or EC2 (long start-up times), in our announcement on the [Actuated blog](https://actuated.com/blog/bringing-firecracker-to-jenkins)

Watch a video walkthrough on our [YouTube channel](https://www.youtube.com/watch?v=LSUVBGfzf3s).

<iframe width="560" height="315" src="https://www.youtube.com/embed/5RjtVM4bvp0?si=LSUVBGfzf3s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## 1. Run a Jenkins master

You can run a Jenkins master within a Slicer microVM, and have everything set up for you automatically via a userdata script.

Create `setup-master.sh`:

```sh
#!/usr/bin/env bash
set -euxo pipefail

# --- Tunables (override via env if you like) ---
: "${ADMIN_USER:=admin}"
: "${ADMIN_PASS:=$(openssl rand -base64 18)}"
: "${JENKINS_HTTP_PORT:=8080}"

# Best-guess URL (update later if you put it behind a domain/reverse-proxy)
DEFAULT_IP="$(hostname -I | awk '{print $1}')"
: "${JENKINS_URL:=http://${DEFAULT_IP}:${JENKINS_HTTP_PORT}/}"

export DEBIAN_FRONTEND=noninteractive

if command -v apt-get >/dev/null 2>&1; then
  # ----- Ubuntu/Debian path -----
  apt-get update && \
    apt-get install -qy --no-install-recommends \
      curl gnupg ca-certificates openjdk-17-jdk git

  # Add Jenkins apt repo
  install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | tee /etc/apt/keyrings/jenkins-keyring.asc >/dev/null
  chmod a+r /etc/apt/keyrings/jenkins-keyring.asc
  echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" > /etc/apt/sources.list.d/jenkins.list

  apt-get update && \
    apt-get install -qy --no-install-recommends \
      jenkins
else
  echo "This setup script targets Debian/Ubuntu (apt). If you need RHEL/Alma/Oracle/Rocky Linux, reach out to us via Discord."
  exit 1
fi

# ----- Plugins configuration -----

# List of plugins to install (name[:version], one per line)
cat > /var/lib/jenkins/plugins.txt <<'EOF'
configuration-as-code
git
workflow-aggregator
credentials
ssh-credentials
EOF

# ----- JCasC config -----
CCFG_DIR="/var/lib/jenkins/casc_configs"
mkdir -p "$CCFG_DIR"
cat > "${CCFG_DIR}/jenkins.yaml" <<'YAML'
jenkins:
  systemMessage: "Jenkins bootstrapped by SlicerVM.com\n"
  mode: EXCLUSIVE
  numExecutors: 0
  remotingSecurity:
    enabled: true
  labelString: "system"
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "${ADMIN_USER}"
          password: "${ADMIN_PASS}"
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false
unclassified:
  location:
    adminAddress: "address not configured yet <nobody@nowhere>"
    url: "${JENKINS_URL}"
YAML

# Substitute vars into JCasC (simple inline envsubst using bash)
sed -i "s|\${ADMIN_USER}|${ADMIN_USER}|g" "${CCFG_DIR}/jenkins.yaml"
sed -i "s|\${ADMIN_PASS}|${ADMIN_PASS}|g" "${CCFG_DIR}/jenkins.yaml"
sed -i "s|\${JENKINS_URL}|${JENKINS_URL}|g" "${CCFG_DIR}/jenkins.yaml"

# Ensure Jenkins uses our JCasC and skips the setup wizard
install -d -m 0755 /etc/systemd/system/jenkins.service.d
cat > /etc/systemd/system/jenkins.service.d/override.conf <<EOF
[Service]
Environment=JENKINS_HOME=/var/lib/jenkins
Environment=CASC_JENKINS_CONFIG=${CCFG_DIR}/jenkins.yaml
Environment=JAVA_OPTS=-Djenkins.install.runSetupWizard=false
Environment=JENKINS_PORT=${JENKINS_HTTP_PORT}
EOF

# Download and use Jenkins Plugin Manager
PLUGIN_MANAGER_JAR="/tmp/jenkins-plugin-manager.jar"
curl -L -o "$PLUGIN_MANAGER_JAR" "https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.13.2/jenkins-plugin-manager-2.13.2.jar"

# Install plugins using the plugin manager
java -jar "$PLUGIN_MANAGER_JAR" -d /var/lib/jenkins/plugins --plugin-file /var/lib/jenkins/plugins.txt --verbose

# Clean up
rm -f "$PLUGIN_MANAGER_JAR"

# Permissions & start
chown -R jenkins:jenkins /var/lib/jenkins
systemctl daemon-reload
systemctl enable --now jenkins

# Wait for it to be reachable
tries=60
until curl -fsS "http://127.0.0.1:${JENKINS_HTTP_PORT}/login" >/dev/null 2>&1 || [ $tries -le 0 ]; do
  sleep 2; tries=$((tries-1))
done

# Save creds somewhere easy to grab
cat > /home/ubuntu/jenkins-admin.txt <<CREDS
Jenkins URL: ${JENKINS_URL}
Username:    ${ADMIN_USER}
Password:    ${ADMIN_PASS}

(saved by user-data at $(date -u +"%Y-%m-%dT%H:%M:%SZ"))
CREDS
sudo chown ubuntu:ubuntu /home/ubuntu/jenkins-admin.txt

echo "======================================================"
echo " Jenkins is up at: ${JENKINS_URL}"
echo " Admin credentials are in: /home/ubuntu/jenkins-admin.txt"
echo "======================================================"
```

You can customise the script as you wish, and the admin password will be printed out to the console during VM creation.

Now create a jenkins-master.yaml file:

```yaml
config:
  host_groups:
  - name: jenkins-master
    userdata_file: ./setup-master.sh
    storage: image
    storage_size: 25G
    count: 1
    vcpu: 2
    ram_gb: 4
    network:
      bridge: brvm0
      tap_prefix: vmtap
      gateway: 192.168.137.1/24

  api:
    enabled: true
    port: 8080
    bind_address: "0.0.0.0:"

  ssh:
    bind_address: "127.0.0.1:"
    port: 2222

  github_user: alexellis

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker
```

The disk size, vCPU and RAM can be adjusted as needed.

You can provide one or more SSH keys via

* `github_user` - your GitHub username, used to fetch your public SSH keys from your profile
* `ssh_keys` - an array of public keys one per line

Now create the VM:

```bash
sudo -E slicer up ./jenkins-master.yaml
```

You can run an interactive shell into the VM and read the password file directly from `/home/ubuntu/jenkins-admin.txt`:

```bash
$ sudo -E ./bin/slicer vm exec jenkins-master-1

Connecting to VM: jenkins-master-1
Connected! Press Ctrl+] to exit.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.10.240 x86_64)

root@jenkins-master-1:/root# cat /home/ubuntu/jenkins-admin.txt

Jenkins URL: http://192.168.137.2:8080/
Username:    admin
Password:    ......
```

Open the URL in a browser and go to the Settings page.

Add any plugins you want, and create jobs as needed.

At a minimum, you should install the Pipeline plugin to allow pipeline jobs to run.

Go to "Manage Jenkins" -> "Manage Plugins" and install the "Pipeline" plugin.

Then, make sure the URL is set for Jenkins itself:

1. Go to "Manage Jenkins" -> "Configure System"
2. Scroll down to "Jenkins Location"
3. Set the "Jenkins URL" to the same URL you used to access Jenkins in your browser
4. Save the configuration.

If you're able, it's also worth disabling the built-in system executors:

1. Go to "Manage Jenkins" -> "Configure System"
2. Scroll down to "Number of executors"
3. Set it to `0`
4. Add a label of "system"
5. Under "Usage" select "Only build jobs with label expressions matching this node"
6. Save the configuration.

This ensures that no jobs run on the master itself, which is a best practice for Jenkins installations.

## 2. Run ephemeral Jenkins build slaves in Slicer via API

In Jenkins, a "Cloud" is simply an API that can add and remove build agents (slaves) on demand.

We built one that can talk to a slicer API endpoint to create and destroy VMs as needed.

Set up another Slicer instance, ideally on another machine on the same network, or with access via an overlay network like a VPN (Wireguard/OpenVPN).

```yaml
config:
  host_groups:
    - name: slave
      storage: zfs
      count: 0
      vcpu: 2
      ram_gb: 8
      network:
        bridge: sbrvm0
        tap_prefix: svmtap
        gateway: 192.168.138.1/24

  ssh_keys:
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCxtkOu6f/UwZP1ITq7dvXUNCX1gZLuf/ReNXEUXFuPumVGuCWVVLQf5zGDKK+BEi0lPBP6F5Pr2XjF84JlQXELkhbp2WczXGzWdZN0EvYcCD3jHk3saeHPK21ndc/on8/FHVZgF5IO444UP1vwpi3jzuNwbxxM3VH1xqH1F/71Zc7VzU+qcsHqA+rWbQX6vFIVhbS9NQKa/OXgxnyTNstTobLMYIy310tDg1/nzhfXK1gmxFTTDlMT2RFi8Gfo/t2pHzKDZCqZnltrcXtrRy9wqKnA6nqJ6dsT3V04IlG+LGTbBGtIxO+x5LQfw0kKnF31uudkL5wgBEgOSK5nwsTp alex@alex-nuc8

  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"

  hypervisor: firecracker
  graceful_shutdown: false

  api:
    enabled: true
    port: 8080
    bind_address: "0.0.0.0:"

  ssh:
    bind_address: "127.0.0.1:"
    port: 2222
```

We recommend that you use [ZFS-backed storage](/docs/storage/zfs) for the slaves because with ZFS, a snapshot is unpacked when Slicer starts up and it can be cloned instantly.

If you don't have time to set up ZFS and are only experimenting, then you can make the following change (but it may add a few seconds to each launch time):

```diff
-      storage: zfs
+      storage: image
+      storage_size: 20G
```

If you're running on the same host as the Jenkins master, ensure that the API and SSH ports don't conflict with the master instance.

```diff
  api:
    enabled: true
+   port: 8081
    bind_address: "0.0.0.0:"

  ssh:
    bind_address: "127.0.0.1:"
+   port: 2223
```

* `count` - set to `0` because slaves are added via the Java Plugin as required
* `storage` - use `zfs` for best performance when starting up new slaves, or `image` for quick experimentation
* `ssh_keys` - add any public keys you want to be able to SSH into the slave VMs - we recommend this over `github_user` since it's much faster than querying GitHub for every launch
* `graceful_shutdown: false` - slaves are ephemeral so we don't need to wait for a graceful shutdown - it will be destroyed immediately when the job is done

### 2.1 Create a custom image for the Jenkins slaves

If you want to pre-install any software on the slave images, you can create your own Dockerfile that extends the base Slicer image.

By pre-loading the JRE, we can speed up Jenkins slave startup times by skipping an installation of Java on every boot. We may produce an image with the JRE pre-installed in a future release, but most teams tend to want to add something like Docker or Terraform into their slave images.

This image is available via CI/CD as: [`ghcr.io/openfaasltd/slicer-slave:latest`](https://ghcr.io/openfaasltd/slicer-slave:latest)

```Dockerfile
FROM ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest

RUN apt-get update -qy && \
  apt-get install -qy  --no-install-recommends \
  curl \
  ca-certificates \
  openjdk-17-jre-headless

RUN useradd -m -s /bin/bash jenkins && \
  echo 'jenkins ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/jenkins && \
  chmod 440 /etc/sudoers.d/jenkins
```

If you also wanted Docker, and basic Kubernetes tooling, you can add the following *extra* lines, which is also available via CI/CD as: [`ghcr.io/openfaasltd/slicer-slave-docker:latest`](https://ghcr.io/openfaasltd/slicer-slave-docker:latest)

```Dockerfile
RUN arkade get \
  --path=/usr/local/bin/ \
  --quiet \
  k3sup \
  kubectl \
  helm \
  kubectx \
  kind

RUN curl -sLS https://get.docker.com | sh && \
  usermod -aG docker jenkins && \
  chmod +x /usr/local/bin/*

RUN systemctl disable regen-ssh-host-keys && \
    systemctl disable ssh && \
    systemctl disable sshd && \
    systemctl disable docker

```

We disable docker in order to make the slave start faster, however Docker can still be activated via systemd's socket activation when needed.

Then build and publish the image to your own container registry.

Edit your `jenkins-slaves.yaml` file to use your custom image:

```diff
-  image: "ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest"
+  image: "ghcr.io/openfaasltd/slicer-slave-docker:latest"
```

Then you only need to relaunch the Slicer slave instance for the new packages to be available.

### 2.2 Start up the Slicer slave instance

Create the Slicer instance for the slaves:

```bash
sudo -E slicer up ./jenkins-slaves.yaml
```

If your master and slave are running on different hosts, make sure you run the routing commands (`ip route add`) that are printed out on start up.

* Master range: 192.168.131.0/24 (255 IP addresses, only one is needed)
* Slave range: 192.168.138.0/24 (255 IP addresses)

If they are both on the same host, then you must not run those commands.

### 2.3 Add the Slicer Cloud to Jenkins

Add the "Slicer VM Cloud" plugin to your Jenkins master via the plugins page.

Upload the `slicer-vm-cloud.hpi` file you received from our team.

Then go to "Manage Jenkins" -> "Configure System" and scroll down to the "Cloud" section.

Add a new Cloud, pick "Slicer VM Cloud".

Enter all the details including the API endpoint for the Slicer instance running the slaves, and its API token (the location is printed upon start-up).

### 2.4 Example Pipeline jobs

For every `pipeline` job you create, you need the following directive:

```
pipeline {
  agent { label 'slicer'}
  options { timeout(time: 2, unit: 'MINUTES') }
```

Now run a quick test job:

```
pipeline {
  agent { label 'slicer'}
  options { timeout(time: 2, unit: 'MINUTES') }
    stages {
    stage('Build') { steps { sh '''
cat /etc/hostname
    ''' } }
  }
}
```

If you see the VM launch, and the runner get created in Jenkins, but it's not able to connect, check the log file for the VM itself under `/var/log/slicer/`. It should be called `slave-1.txt` if it's the first launch so far.

Here's an example that runs a container via Docker:

```
pipeline {
  agent { label 'slicer'}
  options { timeout(time: 2, unit: 'MINUTES') }

  stages {
    stage('Build') { steps { sh '''
sudo systemctl start docker
docker run -i alpine:latest ping -c 4 google.com
    ''' } }
  }
}
```

Another example that installs Kubernetes via K3s and installs OpenFaaS Community Edition (CE):

```
pipeline {
  agent { label 'slicer'}
  options { timeout(time: 2, unit: 'MINUTES') }

  stages {
    stage('Build') { steps { sh '''
export PATH=$PATH:$HOME/.arkade/bin

arkade get k3sup kubectl --progress=false

export KUBECONFIG=`pwd`/kubeconfig

k3sup install --local --no-extras
k3sup ready --attempts 5 --pause 100ms

kubectl get nodes -o wide
kubectl get pods -A -o wide

arkade install openfaas

kubectl get deploy -n openfaas -o wide
''' } }
  }

}
```

The rest is up to you.

### 3. Questions and support

If you have any questions or notice anything unexpected, reach out to us via your support channels. Discord for Home Edition, email for Commercial, and Slack/Email for Enterprise.

**Is the Jenkins URL empty or nil?**

You can then navigate to the Jenkins URL in your web browser to access the Jenkins master using the IP address from the YAML file i.e.

```
http://192.168.137.2:8080/
```

Even if you already see this string, re-enter it and hit Save. Otherwise Jenkins won't be able to generate correct links later on for build slaves.

**Are you having issues with the build slaves?**

Find out whether there are networking or other issues with the following

* Check the logs of the jenkins server on the master

```bash
sudo -E slicer vm exec jenkins-master-1
sudo journalctl -u jenkins.service -f
```

* Check the logs of the slave VM if it is booting

```bash
sudo cat /var/log/slicer/slave-1.txt
```

**Check networking/routing**

If your two Slicer instances are running on different machines, make sure you ran the `ip route` commands as directed.

To verify the routes on either machine use `ping` between the two `.1` IP addresses i.e. `ping 192.168.138.1` from the master host, and `ping 192.168.137.1` from the slave host.

**Check the proper Slicer URL was given for the Slicer Cloud**

So long as you haven't changed anything in the tutorial, the Slicer URL should be `http://192.168.141.1:8082`.

The token is read from `/var/slicer/auth/token`.
