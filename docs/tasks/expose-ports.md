# Expose ports and services from Slicer VMs

There are serveral ways to access services listening on TCP ports within a Slicer VM.

## Direct access via the VM's IP

The easiest way to access a service running on a VM, is to simply use its IP address. This requires any user to be on the same LAN and to have a route to the VM itself.

If you installed the `nginx` package within the VM, you could open a web browser and navigate to `http://192.168.137.2` to access the service.

SSH is already installed on VMs, so if a VM had an IP address of `192.168.137.2`, you could run `ssh ubuntu@192.168.137.2` to access it.

The drawbacks of direct VM IP are that users must have a route to the Slicer host, and the VM's network CIDR. If you're away from the LAN, you will have to use a VPN to get access to the VM which can be complex.

### Port-forwarding over SSH to the VM

SSH tunnelling can be used to access a service that is only listening on 127.0.0.1 within a VM, for instance 127.0.0.1:3000 in the VM, can be accessed via:

```bash
ssh -L 3000:127.0.0.1:3000 ubuntu@192.168.137.2

curl http://localhost:3000
```

## Port forwarding over Slicer's REST API

If you expose or tunnel Slicer's REST API over the Internet using inlets (below), then you can access any service within a VM without any routes or VPNs.

Let's say you've exposed Slicer at `https://slicer-n100-1.example.com` and want to access port 3000 within vm-1:

```bash
slicer vm forward vm-1 127.0.0.1:3000

curl http://localhost:3000
```

You can also remap the local port i.e. from Nginx on 80 to 8080 locally

```bash
slicer vm forward vm-1 8080:127.0.0.1:80

curl http://localhost:8080
```

And you can make forwarded ports available to other machines on your local network this way:

```bash
slicer vm forward vm-1 0.0.0.0:8080:127.0.0.1:80
```

Then you'll be able to access the forwarded service with your own machine's IP address.

## Public access with Inlets

[`inlets-pro`](https://inlets.dev) is a self-hosted tunnel that's easy to use, gives you full privacy and control over security and networking.

You can use inlets-pro or inlets-cloud to expose a service directly from within a VM, or to expose Slicer's REST API for VM management, or port forwarding.

### Expose a TCP service

You can start a TCP tunnel to expose a service at L4. IP whitelists/ACLs can also be added on top, as well as preserving the source IP address via PROXY protocol.

You can set up the `inlets-pro` server manually on a public cloud VM which has a public IP address. Or, for ease of use, `inletsctl` can fully automate the process for you and return a connection string for the client.

TCP tunnels are ideal for exposing things that already have TLS or encryption, or which cannot work over HTTP:

* The Kubernetes API
* SSH
* Reverse proxies like Caddy and Nginx
* Databases (with self-signed certs)

- [Automate a TCP tunnel server](https://docs.inlets.dev/tutorial/automated-tcp-server/)

### Expose a HTTP service with TLS

`inlets-pro` also supports automated TLS termination, and add-on authentication options like static API keys, or OAuth via GitHub or Gmail.

When you expose a local HTTP endpoint i.e. 127.0.0.1:3000 it can be accessed via a DNS record such as `https://wordpress.example.com` and will obtain a TLS certificate from Let's Encrypt.

- [Automate a HTTP tunnel server](https://docs.inlets.dev/tutorial/automated-http-server/)
- [How to authenticate your HTTP tunnels with inlets and OAuth](https://inlets.dev/blog/tutorial/2025/03/10/secure-http-tunnels-with-oauth.html)
- [Authentication options for HTTPS tunnels](https://docs.inlets.dev/tutorial/http-authentication/)

## Inlets Cloud

[Inlets Cloud](https://inlets.dev/cloud) is a completely managed tunnel service available to [inlets-pro](https://inlets.dev) subscribers at no extra cost.

### Expose a HTTP service with TLS and a custom domain

You can create a HTTPS tunnel with a custom domain, and the control plane will terminate TLS for you.

This is the quickest and simplest way to expose a HTTP endpoint on the Internet with TLS.

- [Managed HTTPS tunnels in one-click with inlets cloud](https://inlets.dev/blog/tutorial/2025/04/01/one-click-hosted-http-tunnels.html)

### Expose Kubernetes or SSH for various hosts

Inlets Cloud also supports tunnel that expose Kubernetes or SSH for various hosts, by following a separate guide.

- [SSH Into Any Private Host With Inlets Cloud](https://inlets.dev/blog/tutorial/2024/10/17/ssh-with-inlets-cloud.html)
