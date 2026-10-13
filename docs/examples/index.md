# Examples

These examples show complete setups you can build with Slicer, from sandboxing coding agents to running a home lab DNS server or a Kubernetes cluster. Pick the ones that match what you want to do. You don't need any of the others to get value from Slicer.

Each example has been tested on Slicer for Linux. Most also work on [Slicer for Mac](/mac/overview), but some need adjustments, or need features only available in Slicer for Linux, such as bridge-mode networking or PCI passthrough:

* **Multiple VMs working together**, such as a Kubernetes cluster spread across several VMs. On Slicer for Mac, VMs can't talk to each other directly, so run a single-node cluster instead, or set up an overlay network such as WireGuard through the host. See the [Slicer for Mac overview](/mac/overview).
* **GPUs and other PCI devices**, which are passed through to VMs on Linux hosts only.
* **VMs on your LAN**, such as a router or DNS server that other devices connect to. These rely on Linux networking on the host.

## Sections

* **AI and agents** - run coding agents and local LLMs in isolated VMs.
* **Containers and CI** - Docker, remote builds, Jenkins, and remote editors.
* **Home lab and networking** - a PiHole ad blocker and a Linux router or firewall.
* **Kubernetes** - from a single node to highly available, autoscaling, and multi-machine clusters.
* **OpenFaaS** - serverless functions on K3s, or the all-in-one OpenFaaS Edge.

New to Slicer? Start with the [walkthrough](/getting-started/walkthrough), then come back here.
