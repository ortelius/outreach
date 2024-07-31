- [How to bake an Ortelius Pi Part 3 | Cloudflare](#how-to-bake-an-ortelius-pi-part-3--cloudflare)
  - [Introduction](#introduction)
  - [Roadmap](#roadmap)
  - [Kubernetes](#kubernetes)

## How to bake an Ortelius Pi Part 3 | Cloudflare

### Introduction

In [part 3](https://ortelius.io/blog/2024/04/09/how-to-bake-an-ortelius-pi-part-3--the-gitops-configuration/), of this series we used the [GitOps Methodology](https://gitops.weave.works/) to deploy the [Cert Manager](https://cert-manager.io/), [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised dynamic volume storage, [Metallb Load Balancer](https://metallb.universe.tf/), [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and [Ortelius](https://ortelius.io/) the ultimate evidence store using [Gimlet](https://gimlet.io/) as the UI to our GitOps controller [Fluxcd](https://fluxcd.io/).

In part 4 we will

### Roadmap

I have tried to put things in a logical order for deployment like this:

`cloudflare --> secret store --> ZeroTier --> everything else`

### Kubernetes
