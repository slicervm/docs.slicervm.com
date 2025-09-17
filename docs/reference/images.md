# Images for Slicer MicroVMs

Images for Slicer are built with a Dockerfile, and [can be extended](/tasks/custom-image) with additional layers.

It is also possible to build completely different images from scratch, however this is not recommended or documented at this time. So if you need a different OS, reach out to the team via Discord and request it there.

Cloud Hypervisor is required to mount a PCI device such as an Nvidia GPU into a microVM. In all other cases, Firecracker is preferred.

Generally, Slicer makes use of systemd to run, tune, and manage services and background agents. It is technically possible to use other init systems, but is not supported at this time.

Image availability:

| Operating System | Firecracker (x86_64) | Firecracker (arm64) | Cloud Hypervisor (x86_64) | Cloud Hypervisor (arm64) |
| - | - | - | - | - |
| Ubuntu 22.04 | :white_check_mark: | :white_check_mark: | :white_check_mark: | :cross_mark: |
| Ubuntu 24.04 | :white_check_mark: | :cross_mark: | :cross_mark: | :cross_mark: |
| Rocky Linux 9 | :white_check_mark: | :cross_mark: | :cross_mark: | :cross_mark: |

Ubuntu 22.04 is supported by Cannoncial is supported [until April 2027](https://ubuntu.com/about/release-cycle).

Table of image tags:

| Operating System | Firecracker (x86_64) | Firecracker (arm64) | Cloud Hypervisor (x86_64) | Cloud Hypervisor (arm64) |
| ------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Ubuntu 22.04       | `ghcr.io/openfaasltd/slicer-systemd:5.10.240-x86_64-latest`         | `ghcr.io/openfaasltd/slicer-systemd-arm64:6.1.90-aarch64-latest` | `ghcr.io/openfaasltd/slicer-systemd-ch:5.10.240-x86_64-latest` |  |
| Ubuntu 24.04       | `ghcr.io/openfaasltd/slicer-systemd-2404:5.10.240-x86_64-latest`    |                                                                |                                                              |  |
| Rocky Linux 9      | `ghcr.io/openfaasltd/slicer-systemd-rocky9:5.10.240-x86_64-latest`  |                                                                |                                                              |  |
