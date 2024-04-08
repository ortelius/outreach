## How to bake an Ortelius Pi Part 3 | The Configuration

### Introduction

In [Part 2](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-2-the-preperation/), of this series we configured DHCP, DNS, NFS and deployed MicroK8s. In Part 3 we will deploy the NFS CSI Driver for Kubernetes to connect to the Synology NAS for centralised storage, deploy MetalLB load-balancer, deploy Traefik Proxy as the entrypoint for our Microservices and deploy Ortelius the ultimate evidence store.

### NFS CSI Driver


### MetalLB load-balancer for bare metal Kubernetes

### Traefik the Cloud Native Proxy



### Ortelius the Ultimate Evidence Store




### Conclusion

By this stage you should have three Pi's each ready with the NFS CSI Driver, Traefik and Ortelius up and running. Stay tuned for part 3 where we will deploy the NSF [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes, deploy [Traefik](https://traefik.io/) and [Ortelius](https://ortelius.io/)

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
