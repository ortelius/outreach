- [How to bake an Ortelius Pi Part 4 | Cloudflare, Certificates and Traefik](#how-to-bake-an-ortelius-pi-part-4--cloudflare-certificates-and-traefik)
  - [Introduction](#introduction)
  - [Roadmap](#roadmap)
  - [Cloudflare | Connectivity Cloud](#cloudflare--connectivity-cloud)
    - [Buying a domain name](#buying-a-domain-name)

## How to bake an Ortelius Pi Part 4 | Cloudflare, Certificates and Traefik

### Introduction

In [part 3](https://ortelius.io/blog/2024/04/09/how-to-bake-an-ortelius-pi-part-3--the-gitops-configuration/), of this series we used the [GitOps Methodology](https://opengitops.dev/) to deploy the [Cert Manager](https://cert-manager.io/), [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised dynamic volume storage, [Metallb Load Balancer](https://metallb.universe.tf/), [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and [Ortelius](https://ortelius.io/) the ultimate evidence store using [Gimlet](https://gimlet.io/) as the UI to our GitOps controller [Fluxcd](https://fluxcd.io/).

In part 4 we will setup [Cloudflare](https://www.cloudflare.com/en-gb/), [LetsEncrypt](https://letsencrypt.org/) and [Traefik](https://traefik.io) to secure our connections with certificates.

### Roadmap

I have tried to put things in a logical order for deployment like this:

`cloudflare --> secret store --> ZeroTier --> everything else`

### Cloudflare | Connectivity Cloud

You might know Cloudflare as a CDN but its so much more than that. Cloudflare is packed with amazing features and security offerings which are just to many to cover in this blog.

- [Documentation](https://developers.cloudflare.com/)
- [Learning Centre](https://www.cloudflare.com/en-gb/learning/)
- [Free Plan](https://www.cloudflare.com/plans/free/)

Cloudflare have kindly provided a free plan which we will use so the first thing you need to do is set up an account for yourself or if you have an account login.

#### Buying a domain name

- Now we need a dns domain so if you don't have one you will need to buy one which you can do through Cloudflare.

- Click on `Websites`
![01 cf websites](images/how-to-bake-an-ortelius-pi/part04/01-cf-websites.png)

- Click on `Add a site`
![02 cf add site](images/how-to-bake-an-ortelius-pi/part04/02-cf-add-site.png)

- Click on `register a new domain`
![03 cf register new domain](images/how-to-bake-an-ortelius-pi/part04/03-cf-register-new-domain.png)

- Click in the `Search for a domain name` box and find a domain
![04 cf search domain](images/how-to-bake-an-ortelius-pi/part04/04-cf-search-domain.png)

- Cloudflare will tell you if your domain is available. Unfortunately my cats name `mottles.com` was not available. She will not be impressed
- Pick your domain and brandish your credit card
![05 cf mottles domain](images/how-to-bake-an-ortelius-pi/part04/05-cf-mottles-domain.png)

- At the end of the process you should be able to go back to Websites and see your domain that you registered
- Here you can see my domain `pangarabbit.com`
- Click on our new domain and head over to `DNS`

![06 cf new domain](images/how-to-bake-an-ortelius-pi/part04/06-cf-new-domain.png)

- You should have 2 DNS A records like below | `* is for wildcard` and the domain apex `pangarabbit.com`

![07 cf a records](images/how-to-bake-an-ortelius-pi/part04/08-cf-a-records.png)
