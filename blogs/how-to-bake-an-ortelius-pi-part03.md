## How to bake an Ortelius Pi Part 3 | The Configuration

### Introduction

In [Part 2](https://ortelius.io/blog/2024/04/09/how-to-bake-an-ortelius-pi-part-2-the-configuration/), of this series we deployed DHCP, DNS, NFS (Network File System) storage with a [Synology NAS](https://www.synology.com/) and installed [MicroK8s](https://microk8s.io/) HA cluster.

In part 3 we will use the [GitOps Methodology](https://gitops.weave.works/) to deploy the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised dynamic volume storage, [MetalLB Load Balancer](https://metallb.universe.tf/), [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and [Ortelius](https://ortelius.io/) the ultimate evidence store using [Gimlet](https://gimlet.io/) as the UI to [Fluxcd](https://fluxcd.io/).

### Enter GitOps | Enter Gimlet | Enter Fluxcd

I wanted to find a process for repeatable deployments, and to incorporate drift detection for Kubernetes infrastructure and applications but I was finding it heavy going to use the default values from the providers Helm Chart and then trying to override those with my own values. I couldn't get ArgoCD to do that without some hellish complicated setup until I found Gimlet and Fluxcd which allowed for a single human to have a simple repeatable process.

Gimlet gives us a clean UI for Fluxcd and allows us to have a neat interface into the deployments of our infrastructure and applications. Basically like having the [Little Green Mall Wizard](https://youtu.be/dcxZqMIW4OM) in your K8s cluster with the focus on the wizard part.

### Gimlet

- [Documentation](https://gimlet.io/docs)

Gimlet uses the concepts of Kubernetes Infrastructure and Kubernetes Applications. Infrastructure is the bedrock to deploy applications in an environment such as security, observability, storage, load balancer, proxy and Ortelius. Applications would be the services you provide to end users and customers. This concept is fundamental to understanding the ways of Gimlet and Fluxcd.

Gimlet comes in two flavours [Self-Hosted](https://gimlet.io/docs/installation) and [Cloud hosted](https://accounts.gimlet.io/signup/). I am using Cloud hosted due to the very generous humans at Gimlet.

### Gimlet Installation Self-Hosted

#### Prerequisites

- You will need to have an account with a provider such as [Github](https://github.com/) which is the provider I use as Gimlet is going use this provider for all things GitOps
- [Install Gimlet CLI](https://gimlet.io/docs/installing-gimlet-cli)

```shell
curl -L "https://github.com/gimlet-io/gimlet/releases/download/cli-v0.27.0/gimlet-$(uname)-$(uname -m)" -o gimlet
chmod +x gimlet
sudo mv ./gimlet /usr/local/bin/gimlet
gimlet --version
```

#### Install Gimlet

- Explore more involved installations of Gimlet [here](https://github.com/gimlet-io/gimlet/tree/main/examples)
- We will be using this easy to deploy one-liner for now

```shell
kubectl apply -f https://raw.githubusercontent.com/gimlet-io/gimlet/main/deploy/gimlet.yaml
```

- Then access it with port-forward on http://127.0.0.1:9000

```shell
kubectl port-forward svc/gimlet 9000:9000
```

- Login with Github

![gimlet login](images/how-to-bake-an-ortelius-pi/part03/14-gimlet-login.png)

#### Connect your repositories

- Only connect your application repositories here and not anything to do with infrastructure

![gimlet repos](images/how-to-bake-an-ortelius-pi/part03/15-gimlet-repos.png)

#### Connect you cluster

- Connect your K8s cluster to Gimlet

```shell
gimlet environment connect \
  --env <your environment such as dev or staging or test or prod or anything you like> \
  --server https://app.gimlet.io \
  --token <your token>
```

#### K8s check

- Now if you list your namespaces with the below command you should see `infrastructure`, `flux` and `flux-system`

```shell
kubectl get namespaces
```

- Switch to the namespace `infrastructure`

```shell
kubectl config set-context --current --namespace=infrastructure
```

- List pods and you should see `gimlet-agent-<blah blah blah>`

```shell
kubectl get pods
```

#### Github check

- Go to Github.com and click on your profile in the top right hand corner of your browser tab

![github settings](images/how-to-bake-an-ortelius-pi/part03/16-github-settings.png)

- Scroll down until the left hand coloumn shows `Applications` under the title `Integrations`

![github application](images/how-to-bake-an-ortelius-pi/part03/17-github-application.png)

- You should see the Gimlet application installed
- `!!!WARNING!!!`Whatever you do don't just delete this app like I did and get yourself into account mess hell

![github gimlet app](images/how-to-bake-an-ortelius-pi/part03/18-github-gimlet-app.png)

#### Github Gimlet repo check

- Click on repositories at the top left of the screen
![github gimlet repos](images/how-to-bake-an-ortelius-pi/part03/19-github-gimlet-repos-button.png)

- Then type `gitops-` in the search bar and you should see two repos pop up

![github gimlet repos](images/how-to-bake-an-ortelius-pi/part03/20-github-gimlet-repos.png)

- You should see `gitops-<your environment>-infra` and `gitops-<your environment>-apps`
- We will be spending our time in the infra one for now
- You will also notice that this repo is private thus no one can see any sensitive information such as secrets
- Clone this repo to your local machine

```shell
git clone https://github.com/<your profile>/gitops-<your environment>-infra.git
```

### Gimlet GitOps Infrastructure

#### Kubernetes CSI NFS Driver

With the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) we will use Kubernetes to dynamically manage the creation and mounting of persistent volumes to our pods using the Synology NAS as the central storage server.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Helm Chart reference [here](https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/charts)
- Kubernetes Storage Class docs [here](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [What is network-attached storage (NAS)?](https://www.purestorage.com/knowledge/what-is-nas.html)
- [What is NFS?](https://www.minitool.com/lib/what-is-nfs.html)
- An excellent blog written by Rudi Martinsen on the NFS CSI Driver with step-by-step instructions for reference [here](https://rudimartinsen.com/2024/01/09/nfs-csi-driver-kubernetes/)

### Gimlet Kubernetes CSI NFS Driver deployment

- On your local machine open your IDE and navigate to your cloned infrastructure repo

![github gimlet repos](images/how-to-bake-an-ortelius-pi/part03/21-gimlet-infra.png)

#### Helm-Releases

- A Helm release is an instance of a Helm chart running in a Kubernetes cluster
- Each release is a deployment of a particular version of a chart with a specific configuration
- Create a file called `nfs-csi-driver.yaml` in the `helm-releases` directory and paste the following YAML

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: csi-driver-nfs
  namespace: kube-system # To be installed in the kube-system namespace as required by the csi-driver-nfs
spec:
  interval: 60m
  releaseName: csi-driver-nfs # Helm Chart release name
  chart:
    spec:
      chart: csi-driver-nfs # Name of the Helm Chart
      version: v4.8.0 # Version of the csi-driver-nfs | If a new version comes out simply update here
      sourceRef:
        kind: HelmRepository
        name: csi-driver-nfs
      interval: 10m
  # values: This is where you paste the default values from the Helm Chart provider and configure your overrides for your environment
  values:
    customLabels: {}
    image:
      baseRepo: registry.k8s.io
      nfs:
        repository: registry.k8s.io/sig-storage/nfsplugin
        tag: v4.8.0
        pullPolicy: IfNotPresent
      csiProvisioner:
        repository: registry.k8s.io/sig-storage/csi-provisioner
        tag: v5.0.1
        pullPolicy: IfNotPresent
      csiSnapshotter:
        repository: registry.k8s.io/sig-storage/csi-snapshotter
        tag: v8.0.1
        pullPolicy: IfNotPresent
      livenessProbe:
        repository: registry.k8s.io/sig-storage/livenessprobe
        tag: v2.13.1
        pullPolicy: IfNotPresent
      nodeDriverRegistrar:
        repository: registry.k8s.io/sig-storage/csi-node-driver-registrar
        tag: v2.11.1
        pullPolicy: IfNotPresent
      externalSnapshotter:
        repository: registry.k8s.io/sig-storage/snapshot-controller
        tag: v8.0.1
        pullPolicy: IfNotPresent

    serviceAccount:
      create: true # When true, service accounts will be created for you. Set to false if you want to use your own.
      controller: csi-nfs-controller-sa # Name of Service Account to be created or used
      node: csi-nfs-node-sa # Name of Service Account to be created or used

    rbac:
      create: true
      name: nfs

    driver:
      name: nfs.csi.k8s.io
      mountPermissions: 0

    feature:
      enableFSGroupPolicy: true
      enableInlineVolume: false
      propagateHostMountOptions: false

      kubeletDir: /var/snap/microk8s/common/var/lib/kubelet # This path is specific to MicroK8s

    controller:
      name: csi-nfs-controller
      replicas: 1 # Change if you want more replicas
      strategyType: Recreate
      runOnMaster: false
      runOnControlPlane: false
      livenessProbe:
        healthPort: 29652
      logLevel: 5
      workingMountDir: /tmp
      dnsPolicy: ClusterFirstWithHostNet # available values: Default, ClusterFirstWithHostNet, ClusterFirst
      defaultOnDeletePolicy: delete # available values: delete, retain
      affinity: {}
      nodeSelector: {}
      priorityClassName: system-cluster-critical
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
        - key: "node-role.kubernetes.io/controlplane"
          operator: "Exists"
          effect: "NoSchedule"
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
      resources:
        csiProvisioner:
          limits:
            memory: 400Mi
          requests:
            cpu: 10m
            memory: 20Mi
        csiSnapshotter:
          limits:
            memory: 200Mi
          requests:
            cpu: 10m
            memory: 20Mi
        livenessProbe:
          limits:
            memory: 100Mi
          requests:
            cpu: 10m
            memory: 20Mi
        nfs:
          limits:
            memory: 200Mi
          requests:
            cpu: 10m
            memory: 20Mi

    node:
      name: csi-nfs-node
      dnsPolicy: ClusterFirstWithHostNet # available values: Default, ClusterFirstWithHostNet, ClusterFirst
      maxUnavailable: 1
      logLevel: 5
      livenessProbe:
        healthPort: 29653
      affinity: {}
      nodeSelector: {}
      priorityClassName: system-cluster-critical
      tolerations:
        - operator: "Exists"
      resources:
        livenessProbe:
          limits:
            memory: 100Mi
          requests:
            cpu: 10m
            memory: 20Mi
        nodeDriverRegistrar:
          limits:
            memory: 100Mi
          requests:
            cpu: 10m
            memory: 20Mi
        nfs:
          limits:
            memory: 300Mi
          requests:
            cpu: 10m
            memory: 20Mi

    externalSnapshotter:
      enabled: true
      name: snapshot-controller
      priorityClassName: system-cluster-critical
      controller:
        replicas: 1
      resources:
        limits:
          memory: 300Mi
        requests:
          cpu: 10m
          memory: 20Mi
      # Create volume snapshot CRDs.
      customResourceDefinitions:
        enabled: true #if set true, VolumeSnapshot, VolumeSnapshotContent and VolumeSnapshotClass CRDs will be created. Set it false, If they already exist in cluster.

    ## Reference to one or more secrets to be used when pulling images
    ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
    ##
    imagePullSecrets: []
    # - name: "image-pull-secret"

    # Kubernetes Storage Class creation
    storageClass:
      allowVolumeExpansion: true
      create: true
      name: nfs-csi-default # Give your storage class a meaningful name
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
      provisioner: nfs.csi.k8s.io
      parameters:
        server: 192.168.0.152 # Replace with your NFS server ip address or FQDN
        share: /volume4/pi8s/ # Replace with your NFS share
        #subDir:
        mountPermissions: "0"
        # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
        # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
        # csi.storage.k8s.io/provisioner-secret-namespace: "kube-system"
      reclaimPolicy: Delete
      volumeBindingMode: Immediate
      mountOptions: # This is where you configure your NFS mounts for Linux
        - hard
        - nfsvers=4
```

```shell
kubectl config set-context --current --namespace=kube-system
```

- Helm repo add

```
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```

- Helm repo update

```
helm repo update
```

- Helm repo install

```
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --version v4.6.0 \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set node.dnsPolicy=ClusterFirstWithHostNet \
  --set kubeletDir="/var/snap/microk8s/common/var/lib/kubelet" # The Kubelet has permissions at this location to mount the NFS shares
```

- Kubectl show me the pods

```
kubectl get pods
```

![csi nfs driver storage pods](images/how-to-bake-an-ortelius-pi/part03/01-csi-nfs-driver-pods.png)

- Now lets create a Storage Class to be used for central data access between our nodes and pods
- Create a file called `nfs-setup.yaml`, copy the YAML below and run `kubectl apply -f nfs-setup.yaml`

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

- Great we now have Kubernetes managing NFS volume mounts dynamically for us to our NAS (Network Attached Storage)!

### MetalLB load-balancer for bare metal Kubernetes

With MetalLB we will setup a unique IP address on our home network to expose the Microservices running in our Kubernetes cluster. A public cloud provider would give you this during the deployment of your Kubernetes cluster but since we are the cloud we need to provide it and thats where [MetalLB](https://metallb.universe.tf/) comes in.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/metallb/metallb)
- MetalLB concepts [here](https://metallb.universe.tf/concepts/)

---------------------------------------------------------------------------------------------------------------

- Choose an IP address on your private home network that does not fall inside your DHCP pool for MetalLB to use
- Helm repo add

```
helm repo add metallb https://metallb.github.io/metallb
```

- Helm repo update

```
helm repo update
```

- Helm install MetalLB in the `metallb-system` namespace

```
helm install metallb metallb/metallb -n metallb-system
```

- Kubectl switch to the `metallb-system` namespace

```
kubectl config set-context --current --namespace=metallb-system
```

- Kubectl show me the MetalLB pods in the `metallb-system` namespace

```
kubectl get pods
```

![metallb pods](images/how-to-bake-an-ortelius-pi/part03/03-metallb-pods.png)

- Now lets enable [L2 Advertisement](https://metallb.universe.tf/troubleshooting/) and setup our IP pool
- Copy the YAML below into `metallb-setup.yaml` and run `kubectl apply -f metallb-setup.yaml`

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

- The `ipaddresspools.metallb.io` is a [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) which is a custom resource created in our Kubernetes cluster that adds additional magic
- Kubectl show me all CRDs for MetalLB

```
kubectl get crds | grep metallb
```

![metallb crds](images/how-to-bake-an-ortelius-pi/part03/04-metallb-crds.png)

- Kubectl show me the IP address pools for MetalLB

```
kubectl get ipaddresspools.metallb.io
```

![metallb ip pools](images/how-to-bake-an-ortelius-pi/part03/05-metallb-ip-pool.png)

- Epic we have a working load balancer using a single IP address which will act as a gateway into our Kubernetes cluster which we can control with Traefik Proxy

### Traefik the Cloud Native Proxy

With [Traefik Proxy](https://traefik.io/) we can now direct traffic destined for our Microservices into the Kubernetes cluster and protect our endpoints using a combination of entrypoints, routers, services, providers and middlewares.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Traefik docs [here](https://doc.traefik.io/traefik/)
- Traefik [EntryPoints](https://doc.traefik.io/traefik/routing/entrypoints/)
- Traefik [Routers](https://doc.traefik.io/traefik/routing/routers/)
- Traefik [Services](https://doc.traefik.io/traefik/routing/services/) can be divided into two groups `Traefik Services` and `Kubernetes Services`
- Traefik [Providers](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- Traefik Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/traefik/traefik)

---------------------------------------------------------------------------------------------------------------

- Helm repo add

```
helm repo add traefik https://traefik.github.io/charts
```

- Kubectl create the Traefik namespace

```
kubectl create ns traefik-v2
```

- Kubectl switch to the traefik-v2 namespace

```
kubectl config set-context --current --namespace=traefik-v2
```

- Helm repo update

```
helm repo update
```

- Helm install Traefik

```
helm install traefik traefik/traefik --namespace=traefik-v2
```

- Kubectl show me the pods for Traefik

```
kubectl get pods
```

![traefik pods](images/how-to-bake-an-ortelius-pi/part03/06-traefik-pods.png)

- Using GitHub fork the [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)
- Clone the Helm Chart to your local machine and enable the Traefik `dashboard, kubernetesCRD and kubernetesIngress` in `values.yaml` and don't forget to save
- `FYI` they might already be enabled

```
## Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    # -- Create an IngressRoute for the dashboard
    enabled: true
```

```
providers:
  kubernetesCRD:
    # -- Load Kubernetes IngressRoute provider
    enabled: true
```

```
  kubernetesIngress:
    # -- Load Kubernetes Ingress provider
    enabled: true
```

- Because Traefik is deployed with Helm we will use Helm to update our deployment from `values.yaml`

```
helm upgrade traefik traefik/traefik --values values.yaml
```

- Now we need to deploy an `ingress route` which forms part of the [CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) that were installed with Traefik
- CRDs are custom resources created in our Kubernetes cluster that add additional magic
- Kubectl show me all CRDs for Traefik

```
kubectl get crds | grep traefik
```

![traefik pod](images/how-to-bake-an-ortelius-pi/part03/07-traefik-crds.png)

- Create a file called `dashboard.yaml` and apply the following logic with `kubectl apply -f dashboard.yaml`
- You will need a DNS record created either on your DNS server or in localhosts file to access the dashboard
- Edit localhosts on Linux and Mac here with sudo rights `sudo vi /etc/hosts` by adding `your private ip and traefik.yourdomain.your tld`
- Edit Windows localhosts file here as administrator `windows\System32\drivers\etc\hosts` by adding `your private ip and traefik.yourdomain.your tld`
- [TLD = Top Level Domain](https://en.wikipedia.org/wiki/Top-level_domain)

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

![traefik pod](images/how-to-bake-an-ortelius-pi/part03/08-traefik-ingressroute-dashboard.png)

- Kubectl show me that the Traefik service has claimed our MetalLB single IP address

```
kubectl get svc
```

![traefik service](images/how-to-bake-an-ortelius-pi/part03/09-traefik-service.png)

- Here is a view of the services for all namespaces

```
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
cert-manager     cert-manager                         ClusterIP      10.152.183.42    <none>          9402/TCP                     25h
cert-manager     cert-manager-webhook                 ClusterIP      10.152.183.32    <none>          443/TCP                      25h
default          kubernetes                           ClusterIP      10.152.183.1     <none>          443/TCP                      3d3h
ingress-nginx    ingress-nginx-controller             NodePort       10.152.183.240   <none>          80:31709/TCP,443:30762/TCP   9h
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.152.183.118   <none>          443/TCP                      9h
kube-system      kube-dns                             ClusterIP      10.152.183.10    <none>          53/UDP,53/TCP,9153/TCP       45h
metallb-system   metallb-webhook-service              ClusterIP      10.152.183.117   <none>          443/TCP                      3d2h
netdata          netdata                              ClusterIP      10.152.183.164   <none>          19999/TCP                    2d7h
ortelius         ms-compitem-crud                     NodePort       10.152.183.91    <none>          80:30288/TCP                 3m24s
ortelius         ms-dep-pkg-cud                       NodePort       10.152.183.124   <none>          80:32186/TCP                 3m24s
ortelius         ms-dep-pkg-r                         NodePort       10.152.183.82    <none>          80:31347/TCP                 3m22s
ortelius         ms-general                           NodePort       10.152.183.171   <none>          8080:30704/TCP               3m21s
ortelius         ms-nginx                             NodePort       10.152.183.158   <none>          80:32519/TCP,443:31861/TCP   3m19s
ortelius         ms-postgres                          NodePort       10.152.183.75    <none>          5432:30852/TCP               9h
ortelius         ms-scorecard                         NodePort       10.152.183.74    <none>          80:30674/TCP                 3m18s
ortelius         ms-textfile-crud                     NodePort       10.152.183.200   <none>          80:30126/TCP                 3m16s
ortelius         ms-ui                                NodePort       10.152.183.242   <none>          8080:31073/TCP               3m16s
ortelius         ms-validate-user                     NodePort       10.152.183.55    <none>          80:30266/TCP                 3m16s
traefik-v2       traefik                              LoadBalancer   10.152.183.73    192.168.0.151   80:32700/TCP,443:30988/TCP   2d7h
whoami           whoami                               ClusterIP      10.152.183.168   <none>          80/TCP                       47h```
```

- Brilliant our Traefik Proxy has claimed the IP

What you see is the `traefik` service with the `TYPE LoadBalancer` which means it has claimed the `MetalLB IP` that we assigned. A `CLUSTER-IP` is only accessible inside Kubernetes. So now with MetalLB and Traefik we have built a bridge between the outside world and our internal Kubernetes world. Traefik comes with some self discovery magic in the form of [providers](https://doc.traefik.io/traefik/providers/overview/) which allows Traefik to query `provider` APIs to find relevant information about routing and then dynamically update the routes.

- Hopefully you should be able to access your dashboard at the FQDN (fully qualified domain name) you set

![traefik dashboard](images/how-to-bake-an-ortelius-pi/part03/10-traefik-dashboard.png)

### Ortelius the Ultimate Evidence Store

Well done for making it this far! We have made it to the point where we can deploy Ortelius into our Kubernetes cluster and access Ortelius through the Traefik Proxy inside the Kubernetes Ortelius namespace.

- Kubectl quick reference guide [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Ortelius on GitHub [here](https://github.com/ortelius/)
- Ortelius docs [here](https://docs.ortelius.io/guides/)
- Ortelius Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/ortelius/ortelius)

Ortelius currently consists of the following Microservices. The one we are most interested at this point is `ms-nginx` which is the gateway to all the backing microservices for Ortelius. We are going to deploy Ortelius using Helm then configure Traefik to send requests to `ms-nginx` and then we should get the Ortelius dashboard.

![ortelius microservices](images/how-to-bake-an-ortelius-pi/part03/11-ortelius-microservices.png)

![ortelius dashboard](images/how-to-bake-an-ortelius-pi/part03/12-ortelius-dashboard.png)

##### Ortelius Microservice GitHub repos

- [ms-dep-pkg-cud](https://github.com/ortelius/ms-dep-pkg-cud)
- [ms-textfile-crud](https://github.com/ortelius/ms-textfile-crud)
- [ms-dep-pkg-r](https://github.com/ortelius/ms-dep-pkg-r)
- [ms-compitem-crud](https://github.com/ortelius/ms-compitem-crud)
- [ms-validate-user](https://github.com/ortelius/ms-validate-user)
- [ms-postgres](https://github.com/ortelius/ms-postgres)
- [ms-sbom-export](https://github.com/ortelius/ms-sbom-export)
- [ms-scorecard](https://github.com/ortelius/ms-scorecard)
- [scec-nginx](https://github.com/ortelius/scec-nginx)

---------------------------------------------------------------------------------------------------------------

- Kubectl create the Ortelius namespace

```
kubectl create ns ortelius
```

- Kubectl switch to the ortelius namespace

```
kubectl config set-context --current --namespace=ortelius
```

- Helm repo add

```
helm repo add ortelius https://ortelius.github.io/ortelius-charts/
```

- Helm repo update

```
helm repo update
```

- Helm install Ortelius

```
helm upgrade --install ortelius ortelius/ortelius --set ms-general.dbpass=postgres --set global.postgresql.enabled=true --set global.nginxController.enabled=true --set ms-nginx.ingress.type=k3d --set ms-nginx.ingress.dnsname=<your domain name goes here>  --version "${ORTELIUS_VERSION}" --namespace ortelius
```

-Lets stop here to discuss some of these settings.

- `--set ms-general.dbpass=postgres` | Set the PostgreSQL database password
- `--set global.nginxController.enabled=true` | Sets the ingress controller which could be one of `default nginx ingress, AWS Load Balancer or Google Load Balancer` | Refer to the Helm Chart in ArtifactHub [here](https://artifacthub.io/packages/helm/ortelius/ortelius)
- `--set ms-nginx.ingress.type=k3d` | This setting is for enabling the Traefik Class so that Traefik is made aware of Ortelius even thou its for [K3d](https://k3d.io/v5.6.0/) another very lightweight Kubernetes deployment which uses Traefik as the default ingress
- The `k3d` value enables the Traefik ingress class to make Traefik Ortelius aware.
- `--set ms-nginx.ingress.dnsname=<your domain name goes here>` | This is URL that will go in your browser to access Ortelius

- Kubectl show me the pods for Ortelius

```
kubectl get pods
```

![ortelius microservices](images/how-to-bake-an-ortelius-pi/part03/11-ortelius-microservices.png)

- Now we will deploy a Traefik ingress route for Ortelius by applying the following YAML
- Create a YAML file called `ortelius-traefik.yaml`, copy the YAML into the file and then run `kubectl apply -f ortelius-traefik.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
  labels:
    app: ms-nginx
  name: ms-nginx-traefik
  namespace: ortelius
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: ms-nginx
            port:
              number: 80
        path: /
        pathType: Prefix
status:
  loadBalancer: {}
```

- You should be able to reach the Ortelius dashboard in your browser using the domain name you chose for example mine was `https://ortelius.pangarabbit.com`

Happy alien hunting.......

<img src="images/how-to-bake-an-ortelius-pi/part03/13-ortelius-logo.svg" alt="ortelius-logo" width="600">

### Conclusion

By this stage you should have three Pi's each with the NFS CSI Driver, Traefik and Ortelius up and running. Stay tuned for Part 4 where we unleash GitOps using Gimlet to deploy the following Kubernetes NFS CSI Driver, Metallb load balancer, Traefik Proxy,LetsEncrypt and use Cloudflare to provide email, certificate and TLS services. Yes there is more extraterrestrial life in a cloud deployment near you........

#### Disclaimer: Any brands I mention in this blog post series are not monetised
