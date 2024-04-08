## How to bake an Ortelius Pi Part 3 | The Configuration

### Introduction

In [Part 2](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-2-the-preperation/), of this series we configured DHCP, DNS, NFS and deployed MicroK8s. In Part 3 we will deploy the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised storage, deploy [MetalLB load-balancer](https://metallb.universe.tf/), deploy [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and deploy [Ortelius](https://ortelius.io/) the ultimate evidence store.

We will be using Helm Charts to configure some of our services as this makes getting going a lot easier. Also Helm Charts are great to compare your configuration or reset your `values.yaml` in case you totally lose the plot.

### NFS CSI Driver

With the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) we will use Kubernetes to dynamically manage the creation and mounting of persistent volumes to our pods using the Synology NAS as the central storage server.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Kubernetes Storage Class docs [here](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- An excellent blog written by Rudi Martinsen on the NFS CSI Driver [here](https://rudimartinsen.com/2024/01/09/nfs-csi-driver-kubernetes/)
- Helm Chart reference [here](https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/charts)

- On your local machine open the terminal and use Helm to add the repo and install the driver
- Switch to the `kube-system` namespace

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
  --set kubeletDir="/var/snap/microk8s/common/var/lib/kubelet" # The Kubelet has permissions at this location to mount the NFS shares
```

- Run following to see your pods deployed

```
kubectl get pods
```

![csi nfs driver storage pods](images/how-to-bake-an-ortelius-pi/part03/01-csi-nfs-driver-pods.png)

- Now lets create a Storage Class to be used for central data access between our nodes and pods
- Create a file called `nfs-setup.yaml` and copy the logic below and run `kubectl apply -f nfs-setup.yaml`

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi-default
provisioner: nfs.csi.k8s.io
parameters:
  server: <your nfs server ip goes here>
  share: /volume4/pi8s/
  # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
  # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
  # csi.storage.k8s.io/provisioner-secret-namespace: "default"
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4
```

- Kubectl show me the Storage Class

```
kubectl get sc
```

![csi nfs driver storage class](images/how-to-bake-an-ortelius-pi/part03/02-csi-nfs-driver-storage-class.png)

- Lets make this the default Storage Class as in the above image

```
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- If you want to undo making it the default Storage Class

```
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

- Great we now have Kubernetes managing NFS volume mounts for us!

### MetalLB load-balancer for bare metal Kubernetes

With MetalLB we will setup a unique IP address on our homework to expose the Microservices running in our Kubernetes cluster. A public cloud provider would give you this during the deployment of your Kubernetes cluster but since we are the cloud we need to provide it and that where [MetalLB](https://metallb.universe.tf/) comes in.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Choose an IP address on your private home network that does not fall inside your DHCP pool for MetalLB to use
- Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/metallb/metallb)

- Helm add the repo

```
helm repo add metallb https://metallb.github.io/metallb
```

- Helm install MetalLB in the `metallb-system` namespace

```
helm install metallb metallb/metallb -n metallb-system
```

- Switch to the `metallb-system` namespace

```
kubectl config set-context --current --namespace=metallb-system
```

- Kubernetes show me the MetalLB pods in the `metallb-system` namespace

```
kubectl get pods
```

![metallb pods](images/how-to-bake-an-ortelius-pi/part03/03-metallb-pods.png)

- Now lets enable [L2 Advertisement](https://metallb.universe.tf/troubleshooting/) and setup our IP pool
- Copy this into `metallb-setup.yaml` and run `kubectl apply -f metallb-setup.yaml`

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.151-192.168.0.151 # change this to your private ip
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

- MetalLB show me the IP address pools

```
kubectl get ipaddresspools.metallb.io
```

![metallb ip pools](images/how-to-bake-an-ortelius-pi/part03/04-metallb-ip-pool.png)

- Epic we have a working load balancer for our Kubernetes cluster on a single IP address which means a single gate into our Kubernetes cluster which we can control with Traefik Proxy

### Traefik the Cloud Native Proxy

With [Traefik Proxy](https://traefik.io/) we can now direct traffic destined for our Microservices cluster and protect our endpoints using a combination of routers, services and middlewares.

- Traefik docs [here](https://doc.traefik.io/traefik/)
- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Traefik Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/traefik/traefik)

- Helm add the repo

```
helm repo add traefik https://traefik.github.io/charts
```

- Kubectl create the Traefik namespace

```
kubectl create ns traefik-v2
```

- Switch to the traefik-v2 namespace

```
kubectl config set-context --current --namespace=traefik-v2
```

- Helm install Traefik

```
helm install traefik traefik/traefik --namespace=traefik-v2
```

- Kubectl show me the pods

```
kubectl get pods
```

![traefik pod](images/how-to-bake-an-ortelius-pi/part03/05-traefik-pods.png)

- Using GitHub fork the [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)
- Clone the Helm Chart to your local machine and enable the Traefik dashboard in `values.yaml`
- Don't forget to save

```
## Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    # -- Create an IngressRoute for the dashboard
    enabled: true
```

- Because Traefik is deployed with Helm we will use Helm to update our deployment from `values.yaml`

```
helm upgrade --install traefik traefik/traefik --values values.yaml
```

- Now we need to deploy an `ingress route` which forms part of the [CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) that where installed with Traefik
- CRDs are custom resources created in our Kubernetes cluster that add additional magic
- Create a file called `dashboard.yaml` and apply the following logic with `kubectl apply -f dashboard.yaml`
- You will need a DNS record created either on your DNS server or in localhosts file to access the dashboard
- Edit Mac localhosts file here with sudo rights `sudo vi /etc/hosts` by adding `your private ip and traefik.yourdomain.com`
- Edit Windows localhosts file here with administrartor rights `windows\System32\drivers\etc\hosts` by adding `your private ip and traefik.yourdomain.com`

```
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
  namespace: traefik-v2
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.yourdomain.com`) # This where your DNS records come into play
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```

- Kubectl show me the Traefik ingress routes

```
kubectl get ingressroutes.traefik.io
```

### Ortelius the Ultimate Evidence Store




### Conclusion

By this stage you should have three Pi's each ready with the NFS CSI Driver, Traefik and Ortelius up and running. Stay tuned for Part 3 where we

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
