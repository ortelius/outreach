- [How to bake an Ortelius Pi Part 4 | Cloudflare, Certificates and Traefik](#how-to-bake-an-ortelius-pi-part-4--cloudflare-certificates-and-traefik)
  - [Introduction](#introduction)
  - [Roadmap](#roadmap)
  - [Cloudflare | Connectivity Cloud](#cloudflare--connectivity-cloud)
    - [Buying a domain name](#buying-a-domain-name)
    - [SSL/TLS](#ssltls)

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
![01 cf websites button](images/how-to-bake-an-ortelius-pi/part04/01-cf-websites-button.png)

- Click on `Add a site`
![02 cf add site button](images/how-to-bake-an-ortelius-pi/part04/02-cf-add-site-button.png)

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
![07 cf dns button](images/how-to-bake-an-ortelius-pi/part04/07-cf-dns-button.png)

- You should have 2 DNS A records like below | `* is for wildcard` and the domain apex `pangarabbit.com`
- The domain apex record should be there but you might have to add the wildcard if memory serves me correctly

![08 cf a records](images/how-to-bake-an-ortelius-pi/part04/08-cf-a-records.png)

#### SSL/TLS

- Lets head over to `SSL/TLS` now

![09 cf ssl tls button](images/how-to-bake-an-ortelius-pi/part04/09-cf-ssl-tls-button.png)

- You will be faced with the following screen and you want to have `Full (strict)` enabled

![10 cf ssl tls](images/how-to-bake-an-ortelius-pi/part04/10-cf-ssl-tls.png)

- Below are some caveats to take note of which are taken from that little `Help` button

Why isn’t my site working over HTTPS?

Certificate provisioning typically takes around 15 minutes for paid plans and up to 24 hours for Free plans. Contact support if you do not have a certificate after that time. If the certificate is already “active” under the Edge Certificates tab, but you still cannot access your site over HTTPS, refer to the [troubleshooting documentation](https://developers.cloudflare.com/ssl/troubleshooting/).
What encryption mode should I use?

Cloudflare strongly recommends using Full or Full (strict) modes to prevent malicious connections to your origin. For details on each available mode, refer to the [encryption modes documentation](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/).

- I turned this on

![11 cf ssl tls recommender](images/how-to-bake-an-ortelius-pi/part04/11-cf-ssl-tls-recommender.png)

- Clicking on `Edge Certificates` you will see that the kind folks at Cloudflare have provided you with a certificate for free also known as `Universal SSL`

Attention: Let's Encrypt's chain of trust will be changing on September 2024. Universal SSL certificates will be automatically switched to a more compatible certificate authority. Review our [documentation](https://developers.cloudflare.com/ssl/reference/migration-guides/lets-encrypt-chain/#lets-encrypt-chain-update) for details and to understand the impacts on other certificate types.

![12 cf edge certificates button](images/how-to-bake-an-ortelius-pi/part04/12-cf-edge-certificates-button.png)
![13 cf edge certificates button](images/how-to-bake-an-ortelius-pi/part04/13-cf-edge-certificates.png)
![14 cf edge certificates note](images/how-to-bake-an-ortelius-pi/part04/14-cf-edge-certificates-note.png)

- Enable `Always Use HTTPS`

![15 cf edge certificates https](images/how-to-bake-an-ortelius-pi/part04/15-cf-edge-certificates-https.png)

- I set `Minimum TLS Version` to `TLS 1.3` for the best security

![19 cf edge certificates tls version ](images/how-to-bake-an-ortelius-pi/part04/19-cf-edge-certificates-tls-version.png)

- I enabled `Opportunistic Encryption`

![16 cf edge certificates encryption ](images/how-to-bake-an-ortelius-pi/part04/16-cf-edge-certificates-encryption.png)

- I turned on `TLS 1.3` for the best security

![17 cf edge certificates tls 1.3 ](images/how-to-bake-an-ortelius-pi/part04/17-cf-edge-certificates-tls-1-3.png)

- I turned on `Automatic HTTPS Rewrites`

![18 cf edge certificates https rewrites ](images/how-to-bake-an-ortelius-pi/part04/18-cf-edge-certificates-https-rewrites.png)
