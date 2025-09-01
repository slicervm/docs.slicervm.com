# Introduction to Slicer

Slice up bare-metal into microVMs for R&D, customer support and production workloads.

When commodity clouds are charging [~ 200 USD / mo for as little as 8vCPU and 32GB of RAM](https://blog.alexellis.io/n100-mini-computer/), working at scale becomes prohibitively expensive. You can obtain a brand new mini PC, or used server from eBay for a similar amount as a one-off cost. Or pay a bare-metal cloud provider like Hetzner Robot 33-100 EUR / mo for a powerful server for a public IP with reliable power and Internet. 

Why we built Slicer at OpenFaaS Ltd:

* Customer support and R&D of new versions of software.
* Kubernetes has a node limit of around 100 Pods - making it wasteful on larger machines.
* Cloud VMs are exorbitantly priced, with limited resources
* Mini PCs, old desktops, servers from eBay and bare-metal from Hetzner can not just run, but outpace many cloud VMs often with super fast NVMe storage.

Learn more in Alex's blog post [Preview: Slice Up Bare-Metal with Slicer](https://blog.alexellis.io/slicer-bare-metal-preview/).

See [initial customer interest via this X/Twitter post](https://x.com/alexellisuk/status/1961752898552914074)

![Slicer running on a Raspberry Pi 5 with NVMe](images/rpi5.png)
> Slicer running on a Raspberry Pi 5 with NVMe

## Use-cases

### Super fast Kubernetes at scale

Reproducing a customer support case with Kubernetes needing 7000 Pods. The time to create a 3-node cluster on AWS EKS is approximately 30 minutes. With Slicer, this can be reduced to single digit minutes.

### Chaos testing & customer support

When writing Kubernetes operators, failure conditions are often overlooked. And no, you don't need a special framework on the CNCF landscape to do this. Slicer's Serial Over SSH console means you can continue working on a machine after taking down the network.

### Great value production

The first way we deployed Slicer into a permanent setup was by taking a Hetzner host with a 8-core AMD Ryzen and 64GB of RAM, and local NVMe. We created a HA, 3-node K3s cluster capable of running 300 Pods and deployed our long term testing environments to it, later adding production workloads SaaS like Inlets Cloud.

### AI and the agentic future

Slicer isn't tied to Firecracker. With the Cloud Hypervisor support, any kind of GPU from an Nvidia 3090 RTX to a Tesla A100 can be used run local LLMs using Ollama, or a tool of your choice.

Ephemeral VMs can be launched for agents via API, or command line - supplying a userdata script for bootstrap. You can then set up your own agent for command & control.

### Better than containers

Docker is convenient and ubiquitous, but if you're having to give it all kinds of privileged flags just to make something work like Wireguard or containerd, then you're doing it all wrong.

A microVM gives you an entire, real guest with its own Kernel.

Our second production instance of Slicer has three VMs in its group: 1 hosts a Ghost blog using Docker, 2 hosts the control-plane for Actuated along with all its requisite storage and state, 3 hosts an OpenFaaS Edge instance running in containerd. Everything is completely isolated and portable.

## Next steps

* [Install Slicer](/getting-started/install) on a spare machine, old laptop or mini PC, etc.
* [Read the launch blog post](https://blog.alexellis.io/slicer-bare-metal-preview/)

Or, [get in touch](/contact).

