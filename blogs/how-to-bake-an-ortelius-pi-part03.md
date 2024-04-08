## How to bake an Ortelius Pi Part 3 | The Configuration

### Introduction

In [Part 2](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-2-the-preperation/), of this series we configured DHCP, DNS, NFS and deployed MicroK8s. In Part 3 we will deploy the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised storage, deploy [MetalLB load-balancer](https://metallb.universe.tf/), deploy [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and deploy [Ortelius](https://ortelius.io/) the ultimate evidence store.

### NFS CSI Driver
- [Kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm Chart reference [here](https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/charts)
- On your local machine open the terminal and use Helm to add the repo and install the driver

- Run following to switch to the kube-system namespace
```
kubectl config set-context --current --namespace=kube-system
```
```
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```
```
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --version v4.6.0 \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set node.dnsPolicy=ClusterFirstWithHostNet \
  --set kubeletDir="/var/snap/microk8s/common/var/lib/kubelet"
```
- The K8s Kubelet will mount the shares for us










### MetalLB load-balancer for bare metal Kubernetes

### Traefik the Cloud Native Proxy



### Ortelius the Ultimate Evidence Store




### Conclusion

By this stage you should have three Pi's each ready with the NFS CSI Driver, Traefik and Ortelius up and running. Stay tuned for Part 3 where we

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
