## How to bake an Ortelius Pi Part 3 | The GitOps Configuration

### Introduction

In [Part 2](https://ortelius.io/blog/2024/04/09/how-to-bake-an-ortelius-pi-part-2-the-configuration/), of this series we deployed DHCP, DNS, NFS (Network File System) storage with a [Synology NAS](https://www.synology.com/) and installed [MicroK8s](https://microk8s.io/) HA cluster.

In part 3 we will use the [GitOps Methodology](https://gitops.weave.works/) to deploy the [NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) for Kubernetes to connect to the Synology NAS for centralised dynamic volume storage, [MetalLB Load Balancer](https://metallb.universe.tf/), [Traefik Proxy](https://traefik.io/) as the entrypoint for our Microservices and [Ortelius](https://ortelius.io/) the ultimate evidence store using [Gimlet](https://gimlet.io/) as the UI to [Fluxcd](https://fluxcd.io/).

### Enter GitOps | Enter Gimlet | Enter Fluxcd

I wanted to find a process for repeatable deployments, and to incorporate drift detection for Kubernetes infrastructure and applications but I was finding it heavy going to use the default values from the providers Helm Chart and then trying to override those with my own values. I couldn't get ArgoCD to do that without some hellish complicated setup until I found Gimlet and Fluxcd which allowed for a single human to have a simple repeatable process.

Gimlet gives us a clean UI for Fluxcd and allows us to have a neat interface into the deployments of our infrastructure and applications. Basically like having the [Little Green Mall Wizard](https://youtu.be/dcxZqMIW4OM) in your K8s cluster with the focus on the wizard part.

### Gimlet

- [Documentation](https://gimlet.io/docs)
- [Managing infrastructure components](https://gimlet.io/docs/managing-infrastructure-components)
- [On the command line](https://gimlet.io/docs/managing-infrastructure-components#on-the-command-line)
- [Gimlet manifest reference](https://gimlet.io/docs/gimlet-manifest-reference)
- [Gimlet OneChart reference](https://gimlet.io/docs/onechart-reference)
- [Gimlet configuration reference](https://gimlet.io/docs/gimlet-configuration-reference)
- [Upgrading Flux](https://gimlet.io/docs/gitops-bootstrapping-reference)

Gimlet uses the concepts of Kubernetes Infrastructure and Kubernetes Applications. Infrastructure is the bedrock to deploy applications in an environment such as security, observability, storage, load balancer, proxy and Ortelius. Applications would be the services you provide to end users and customers. This concept is fundamental to understanding the ways of Gimlet and Fluxcd.

Gimlet comes in two flavours [Self-Hosted](https://gimlet.io/docs/installation) and [Cloud hosted](https://accounts.gimlet.io/signup/). I am using Cloud hosted due to the very generous humans at Gimlet.

#### Fluxcd

- [Documentation](https://fluxcd.io/flux/)
- [Flux CLI](https://fluxcd.io/flux/cmd/)
- [Flux Ecosystem](https://fluxcd.io/ecosystem/#flux-uis--guis)
- [VS Code extension](https://marketplace.visualstudio.com/items?itemName=Weaveworks.vscode-gitops-tools)

![fluxcd vscode extension](images/how-to-bake-an-ortelius-pi/part03/24-fluxcd-vscode-extension.png)

- `kubectl get crds | grep flux`
![fluxcd crds](images/how-to-bake-an-ortelius-pi/part03/25-fluxcd-crds.png)

### Gimlet Installation Self-Hosted

#### Prerequisites

- You will need to have an account with a provider such as [Github](https://github.com/) which is the provider I use
- Gimlet is going to use this provider for all things GitOps
- [Install Gimlet CLI](https://gimlet.io/docs/installing-gimlet-cli)

```shell
# Check the current version before you install
curl -L "https://github.com/gimlet-io/gimlet/releases/download/cli-v0.27.0/gimlet-$(uname)-$(uname -m)" -o gimlet
chmod +x gimlet
sudo mv ./gimlet /usr/local/bin/gimlet
gimlet --version
```

#### Gimlet on the command line

- `FYI` Please read this [On the command line](https://gimlet.io/docs/managing-infrastructure-components#on-the-command-line)
- We will be spending all of our time in the `gitops-<your environment>-infra` repo to deploy our Kuberenetes infrastructure with Gimlet

![gimlet infra](images/how-to-bake-an-ortelius-pi/part03/21-gimlet-infra.png)

#### Install Gimlet

- Explore more involved installations of Gimlet [here](https://github.com/gimlet-io/gimlet/tree/main/examples)
- We will be using this easy to deploy one-liner for now

```shell
kubectl apply -f https://raw.githubusercontent.com/gimlet-io/gimlet/main/deploy/gimlet.yaml
```

- Then access it with port-forward on <http://127.0.0.1:9000>

```shell
kubectl port-forward svc/gimlet 9000:9000
```

- Login with Github

![gimlet login](images/how-to-bake-an-ortelius-pi/part03/14-gimlet-login.png)

#### Connect your repositories

- Only import your application repositories here and not anything to do with infrastructure

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

- Go to [Github.com](https://github.com/) and click on your profile in the top right hand corner of your browser tab

![github settings](images/how-to-bake-an-ortelius-pi/part03/16-github-settings.png)

- Scroll down until the left hand coloumn shows `Applications` under the title `Integrations`

![github application](images/how-to-bake-an-ortelius-pi/part03/17-github-application.png)

- You should see the Gimlet application installed
- `!!!WARNING!!!`Whatever you do don't just delete this app like I did and get yourself into account mess

![github gimlet app](images/how-to-bake-an-ortelius-pi/part03/18-github-gimlet-app.png)

#### Github Gimlet repo check

- Click on repositories at the top left of the screen
![github gimlet repos](images/how-to-bake-an-ortelius-pi/part03/19-github-gimlet-repos-button.png)

- Then type `gitops-` in the search bar and you should see two repos pop up

![github gimlet repos](images/how-to-bake-an-ortelius-pi/part03/20-github-gimlet-repos.png)

- You should see `gitops-<your environment>-infra` and `gitops-<your environment>-apps`
- We will be spending our time in the infra one for now
- You will also notice that this repo is private thus no one can see any sensitive information such as secrets
- I will be including [Doppler](https://www.doppler.com/) later for secrets management
- Clone this repo to your local machine

- Gimlet Gitops Infra

![gimlet infra repo](images/how-to-bake-an-ortelius-pi/part03/22-gimlet-infra-repo.png)

- Gimlet Gitops Apps

![gimlet apps repo](images/how-to-bake-an-ortelius-pi/part03/23-gimlet-apps-repo.png)

```shell
git clone https://github.com/<your profile>/gitops-<your environment>-infra.git
```

```shell
git clone https://github.com/<your profile>/gitops-<your environment>-apps.git
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

#### Helm-Repository | CSI NFS Driver

- Lets add the Kubernetes CSI NFS Driver Helm repository
- A Helm repository is a collection of Helm charts that are made available for download and installation
- Helm repositories serve as centralised locations where Helm charts can be stored, shared, and managed
- Create a file called `nfs-csi-driver.yaml` in the `helm-repositories` directory and paste the following YAML

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: csi-driver-nfs
  namespace: kube-system
spec:
  interval: 60m
  url: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```

#### Helm-Release | CSI NFS Driver

- Lets create a Helm release of the Kubernetes CSI NFS Driver
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

      kubeletDir: /var/snap/microk8s/common/var/lib/kubelet # This path is specific to MicroK8s as per the documentation

    controller:
      name: csi-nfs-controller
      replicas: 3 # Change amount of replicas
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
        storageclass.kubernetes.io/is-default-class: "true" # Sets this Storage Class as the default
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

- Lets git it

```shell
git add .
git commit -m "k8s infra csi nfs driver deploy"
git push
```

#### Fluxcd is doing the following under the hood

- Helm repo add

```shell
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
```

- Helm repo update

```shell
helm repo update
```

- Helm repo install

```shell
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --version v4.8.0 \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set node.dnsPolicy=ClusterFirstWithHostNet \
  --set kubeletDir="/var/snap/microk8s/common/var/lib/kubelet"
```

#### Kubernetes check

- Kubectl switch to the `kube-system` namespace

```shell
kubectl config set-context --current --namespace=kube-system
```

- Kubectl show me the pods

```shell
kubectl get pods
```

![csi nfs driver storage pods](images/how-to-bake-an-ortelius-pi/part03/01-csi-nfs-driver-pods.png)

- Kubectl show me the Storage Class

```shell
kubectl get sc
```

![csi nfs driver storage class](images/how-to-bake-an-ortelius-pi/part03/02-csi-nfs-driver-storage-class.png)

- From CSI NFS Driver version v4.8.0 you no longer have to manually set the default Storage Class as there is an annotation provided

```yaml
      annotations:
        storageclass.kubernetes.io/is-default-class: "true" # Sets this Storage Class as the default
```

- Manually setting and unsetting the default Storage Class

```shell
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```shell
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

- Great we now have Kubernetes managing NFS volume mounts dynamically!

#### Manifest Folder | CSI NFS Driver

- I am currently using the manifests folder for manual Kubernetes manifest deploys

### MetalLB load-balancer for bare metal Kubernetes

With MetalLB we will setup a unique IP address on our home network to expose the Microservices running in our Kubernetes cluster. A public cloud provider would give you this during the deployment of your Kubernetes cluster but since we are the cloud we need to provide it and thats where [MetalLB](https://metallb.universe.tf/) comes in.

- Kubectl quick reference [here](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Helm cheat sheet [here](https://helm.sh/docs/intro/cheatsheet/)
- Helm Chart on ArtifactHub [here](https://artifacthub.io/packages/helm/metallb/metallb)
- MetalLB concepts [here](https://metallb.universe.tf/concepts/)

#### Helm-Repository | Metallb

- Lets add the Metallb Helm repository
- A Helm repository is a collection of Helm charts that are made available for download and installation
- Helm repositories serve as centralised locations where Helm charts can be stored, shared, and managed
- Create a file called `metallb.yaml` in the `helm-repositories` directory and paste the following YAML
- Choose an IP address on your private home network that does not fall inside your DHCP pool for MetalLB to use

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: metallb
  namespace: infrastructure
spec:
  interval: 60m
  url: https://metallb.github.io/metallb
```

#### Helm-Release | Metallb

- Lets create a Helm release for Metallb
- A Helm release is an instance of a Helm chart running in a Kubernetes cluster
- Each release is a deployment of a particular version of a chart with a specific configuration
- Create a file called `metallb.yaml` in the `helm-releases` directory and paste the following YAML

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: metallb
  namespace: infrastructure
spec:
  interval: 60m
  releaseName: metallb
  chart:
    spec:
      chart: metallb
      version: v0.14.8
      sourceRef:
        kind: HelmRepository
        name: metallb
      interval: 10m
  values:
    # Default values for metallb.
    # This is a YAML-formatted file.
    # Declare variables to be passed into your templates.

    imagePullSecrets: []
    nameOverride: ""
    fullnameOverride: ""
    loadBalancerClass: ""

    # To configure MetalLB, you must specify ONE of the following two
    # options.

    rbac:
      # create specifies whether to install and use RBAC rules.
      create: true

    prometheus:
      # scrape annotations specifies whether to add Prometheus metric
      # auto-collection annotations to pods. See
      # https://github.com/prometheus/prometheus/blob/release-2.1/documentation/examples/prometheus-kubernetes.yml
      # for a corresponding Prometheus configuration. Alternatively, you
      # may want to use the Prometheus Operator
      # (https://github.com/coreos/prometheus-operator) for more powerful
      # monitoring configuration. If you use the Prometheus operator, this
      # can be left at false.
      scrapeAnnotations: false

      # port both controller and speaker will listen on for metrics
      metricsPort: 7472

      # if set, enables rbac proxy on the controller and speaker to expose
      # the metrics via tls.
      # secureMetricsPort: 9120

      # the name of the secret to be mounted in the speaker pod
      # to expose the metrics securely. If not present, a self signed
      # certificate to be used.
      speakerMetricsTLSSecret: ""

      # the name of the secret to be mounted in the controller pod
      # to expose the metrics securely. If not present, a self signed
      # certificate to be used.
      controllerMetricsTLSSecret: ""

      # prometheus doens't have the permission to scrape all namespaces so we give it permission to scrape metallb's one
      rbacPrometheus: true

      # the service account used by prometheus
      # required when " .Values.prometheus.rbacPrometheus == true " and " .Values.prometheus.podMonitor.enabled=true or prometheus.serviceMonitor.enabled=true "
      serviceAccount: ""

      # the namespace where prometheus is deployed
      # required when " .Values.prometheus.rbacPrometheus == true " and " .Values.prometheus.podMonitor.enabled=true or prometheus.serviceMonitor.enabled=true "
      namespace: ""

      # the image to be used for the kuberbacproxy container
      rbacProxy:
        repository: gcr.io/kubebuilder/kube-rbac-proxy
        tag: v0.12.0
        pullPolicy:

      # Prometheus Operator PodMonitors
      podMonitor:
        # enable support for Prometheus Operator
        enabled: false

        # optional additionnal labels for podMonitors
        additionalLabels: {}

        # optional annotations for podMonitors
        annotations: {}

        # Job label for scrape target
        jobLabel: "app.kubernetes.io/name"

        # Scrape interval. If not set, the Prometheus default scrape interval is used.
        interval:

        # 	metric relabel configs to apply to samples before ingestion.
        metricRelabelings: []
        # - action: keep
        #   regex: 'kube_(daemonset|deployment|pod|namespace|node|statefulset).+'
        #   sourceLabels: [__name__]

        # 	relabel configs to apply to samples before ingestion.
        relabelings: []
        # - sourceLabels: [__meta_kubernetes_pod_node_name]
        #   separator: ;
        #   regex: ^(.*)$
        #   target_label: nodename
        #   replacement: $1
        #   action: replace

      # Prometheus Operator ServiceMonitors. To be used as an alternative
      # to podMonitor, supports secure metrics.
      serviceMonitor:
        # enable support for Prometheus Operator
        enabled: false

        speaker:
          # optional additional labels for the speaker serviceMonitor
          additionalLabels: {}
          # optional additional annotations for the speaker serviceMonitor
          annotations: {}
          # optional tls configuration for the speaker serviceMonitor, in case
          # secure metrics are enabled.
          tlsConfig:
            insecureSkipVerify: true

        controller:
          # optional additional labels for the controller serviceMonitor
          additionalLabels: {}
          # optional additional annotations for the controller serviceMonitor
          annotations: {}
          # optional tls configuration for the controller serviceMonitor, in case
          # secure metrics are enabled.
          tlsConfig:
            insecureSkipVerify: true

        # Job label for scrape target
        jobLabel: "app.kubernetes.io/name"

        # Scrape interval. If not set, the Prometheus default scrape interval is used.
        interval:

        # 	metric relabel configs to apply to samples before ingestion.
        metricRelabelings: []
        # - action: keep
        #   regex: 'kube_(daemonset|deployment|pod|namespace|node|statefulset).+'
        #   sourceLabels: [__name__]

        # 	relabel configs to apply to samples before ingestion.
        relabelings: []
        # - sourceLabels: [__meta_kubernetes_pod_node_name]
        #   separator: ;
        #   regex: ^(.*)$
        #   target_label: nodename
        #   replacement: $1
        #   action: replace

      # Prometheus Operator alertmanager alerts
      prometheusRule:
        # enable alertmanager alerts
        enabled: false

        # optional additionnal labels for prometheusRules
        additionalLabels: {}

        # optional annotations for prometheusRules
        annotations: {}

        # MetalLBStaleConfig
        staleConfig:
          enabled: true
          labels:
            severity: warning

        # MetalLBConfigNotLoaded
        configNotLoaded:
          enabled: true
          labels:
            severity: warning

        # MetalLBAddressPoolExhausted
        addressPoolExhausted:
          enabled: true
          labels:
            severity: alert

        addressPoolUsage:
          enabled: true
          thresholds:
            - percent: 75
              labels:
                severity: warning
            - percent: 85
              labels:
                severity: warning
            - percent: 95
              labels:
                severity: alert

        # MetalLBBGPSessionDown
        bgpSessionDown:
          enabled: true
          labels:
            severity: alert

        extraAlerts: []

    # controller contains configuration specific to the MetalLB cluster
    # controller.
    controller:
      enabled: true
      # -- Controller log level. Must be one of: `all`, `debug`, `info`, `warn`, `error` or `none`
      logLevel: info
      # command: /controller
      # webhookMode: enabled
      image:
        repository: quay.io/metallb/controller
        tag:
        pullPolicy:
      ## @param controller.updateStrategy.type Metallb controller deployment strategy type.
      ## ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
      ## e.g:
      ## strategy:
      ##  type: RollingUpdate
      ##  rollingUpdate:
      ##    maxSurge: 25%
      ##    maxUnavailable: 25%
      ##
      strategy:
        type: RollingUpdate
      serviceAccount:
        # Specifies whether a ServiceAccount should be created
        create: true
        # The name of the ServiceAccount to use. If not set and create is
        # true, a name is generated using the fullname template
        name: ""
        annotations: {}
      securityContext:
        runAsNonRoot: true
        # nobody
        runAsUser: 65534
        fsGroup: 65534
      resources:
        {}
        # limits:
        # cpu: 100m
        # memory: 100Mi
      nodeSelector: {}
      tolerations: []
      priorityClassName: ""
      runtimeClassName: ""
      affinity: {}
      podAnnotations: {}
      labels: {}
      livenessProbe:
        enabled: true
        failureThreshold: 3
        initialDelaySeconds: 10
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 1
      readinessProbe:
        enabled: true
        failureThreshold: 3
        initialDelaySeconds: 10
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 1
      tlsMinVersion: "VersionTLS12"
      tlsCipherSuites: ""

      extraContainers: []

    # speaker contains configuration specific to the MetalLB speaker
    # daemonset.
    speaker:
      enabled: true
      # command: /speaker
      # -- Speaker log level. Must be one of: `all`, `debug`, `info`, `warn`, `error` or `none`
      logLevel: info
      tolerateMaster: true
      memberlist:
        enabled: true
        mlBindPort: 7946
        mlBindAddrOverride: ""
        mlSecretKeyPath: "/etc/ml_secret_key"
      excludeInterfaces:
        enabled: true
      # ignore the exclude-from-external-loadbalancer label
      ignoreExcludeLB: false

      image:
        repository: quay.io/metallb/speaker
        tag:
        pullPolicy:
      ## @param speaker.updateStrategy.type Speaker daemonset strategy type
      ## ref: https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/
      ##
      updateStrategy:
        ## StrategyType
        ## Can be set to RollingUpdate or OnDelete
        ##
        type: RollingUpdate
      serviceAccount:
        # Specifies whether a ServiceAccount should be created
        create: true
        # The name of the ServiceAccount to use. If not set and create is
        # true, a name is generated using the fullname template
        name: ""
        annotations: {}
      securityContext: {}
      ## Defines a secret name for the controller to generate a memberlist encryption secret
      ## By default secretName: {{ "metallb.fullname" }}-memberlist
      ##
      # secretName:
      resources:
        {}
        # limits:
        # cpu: 100m
        # memory: 100Mi
      nodeSelector: {}
      tolerations: []
      priorityClassName: ""
      affinity: {}
      ## Selects which runtime class will be used by the pod.
      runtimeClassName: ""
      podAnnotations: {}
      labels: {}
      livenessProbe:
        enabled: true
        failureThreshold: 3
        initialDelaySeconds: 10
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 1
      readinessProbe:
        enabled: true
        failureThreshold: 3
        initialDelaySeconds: 10
        periodSeconds: 10
        successThreshold: 1
        timeoutSeconds: 1
      startupProbe:
        enabled: true
        failureThreshold: 30
        periodSeconds: 5
      # frr contains configuration specific to the MetalLB FRR container,
      # for speaker running alongside FRR.
      frr:
        enabled: true
        image:
          repository: quay.io/frrouting/frr
          tag: 9.1.0
          pullPolicy:
        metricsPort: 7473
        resources: {}

        # if set, enables a rbac proxy sidecar container on the speaker to
        # expose the frr metrics via tls.
        # secureMetricsPort: 9121

      reloader:
        resources: {}

      frrMetrics:
        resources: {}

      extraContainers: []

    crds:
      enabled: true
      validationFailurePolicy: Fail

    # frrk8s contains the configuration related to using an frrk8s instance
    # (github.com/metallb/frr-k8s) as the backend for the BGP implementation.
    # This allows configuring additional frr parameters in combination to those
    # applied by MetalLB.
    frrk8s:
      # if set, enables frrk8s as a backend. This is mutually exclusive to frr
      # mode.
      enabled: false
      external: false
      namespace: ""
```

- Lets git it

```shell
git add .
git commit -m "k8s infra metallb"
git push
```

#### Fluxcd is doing the following under the hood

- Helm repo add

```shell
helm repo add metallb https://metallb.github.io/metallb
```

- Helm repo update

```shell
helm repo update
```

- Helm install MetalLB in the `infrastructure` namespace

```shell
helm install metallb metallb/metallb -n infrastructure
```

#### Kubernetes check

- Kubectl switch to the `infrastructure` namespace

```shell
kubectl config set-context --current --namespace=infrastructure
```

- Kubectl show me the MetalLB pods in the `infrastructure` namespace

```shell
kubectl get pods
```

![metallb pods](images/how-to-bake-an-ortelius-pi/part03/03-metallb-pods.png)

#### Manifest Folder | Metallb

- I am currently using the manifests folder for manual Kubernetes manifest deploys
- Now lets enable [L2 Advertisement](https://metallb.universe.tf/troubleshooting/) and setup our IP pool
- Copy the YAML below into `metallb.yaml` and run `kubectl apply -f metallb.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: infrastructure
spec:
  addresses:
  - 192.168.0.151-192.168.0.151 # Amend this to match your private ip address pool
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-pool
  namespace: infrastructure
spec:
  ipAddressPools:
  - default-pool
```

- The `ipaddresspools.metallb.io` is a [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) which is a custom resource created in our Kubernetes cluster that adds additional functionality
- Kubectl show me all CRDs for MetalLB

```shell
kubectl get crds | grep metallb
```

![metallb crds](images/how-to-bake-an-ortelius-pi/part03/04-metallb-crds.png)

- Kubectl show me the IP address pools for MetalLB

```shell
kubectl get ipaddresspools.metallb.io
```

![metallb ip pools](images/how-to-bake-an-ortelius-pi/part03/05-metallb-ip-pool.png)

- Epic we have a working load balancer using a single IP address which will act as a gateway into our Kubernetes cluster which we can control with Traefik Proxy and which Traefik Proxy can bind to

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

#### Helm-Repository | Traefik

- Lets add the Traefik Helm repository
- A Helm repository is a collection of Helm charts that are made available for download and installation
- Helm repositories serve as centralised locations where Helm charts can be stored, shared, and managed
- Create a file called `traefik.yaml` in the `helm-repositories` directory and paste the following YAML
- Choose an IP address on your private home network that does not fall inside your DHCP pool for MetalLB to use

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: traefik
  namespace: infrastructure
spec:
  interval: 60m
  url: https://traefik.github.io/charts
```

#### Helm-Release | Traefik

- Lets create a Helm release for Metallb
- A Helm release is an instance of a Helm chart running in a Kubernetes cluster
- Each release is a deployment of a particular version of a chart with a specific configuration
- Create a file called `traefik.yaml` in the `helm-releases` directory and paste the following YAML

#### FYI | Helm Chart configuration | Config changes you need to make to Traefik to fit your environment

```yaml
deployment:
  replicas: 3 # Maybe you only want 1
```

```yaml
    # The gatekeeper to your Microservices
    ingressRoute:
      dashboard:
        # -- Create an IngressRoute for the dashboard
        enabled: true
        # -- Additional ingressRoute annotations (e.g. for kubernetes.io/ingress.class)
        annotations: {}
        # -- Additional ingressRoute labels (e.g. for filtering IngressRoute by custom labels)
        labels: {}
        # -- The router match rule used for the dashboard ingressRoute
        matchRule: Host(`<add your fqdn here>`) #PathPrefix(`/dashboard`) || PathPrefix(`/api`)
        # -- Specify the allowed entrypoints to use for the dashboard ingress route, (e.g. traefik, web, websecure).
        # By default, it's using traefik entrypoint, which is not exposed.
        # /!\ Do not expose your dashboard without any protection over the internet /!\
        entryPoints: ["web", "websecure"]
        # -- Additional ingressRoute middlewares (e.g. for authentication)
```

#### Manifest Folder | Traefik

- The folks at Traefik put this nice piece of logic in the Helm Chart that allows you to create a config file which is dynamically monitored by Traefik
- I used the config file to manage the Lets Encrypt certicate renewal in conjunction with Cloudflare
- I have `DISABLED` this logic in the below Helm Chart values config

```yaml
      file:
        # -- Create a file provider
        enabled: false
        # -- Allows Traefik to automatically watch for file changes
        watch: true
        # -- File content (YAML format, go template supported) (see https://doc.traefik.io/traefik/providers/file/)
        # content:
        providers:
          file:
            directory: /manifests/traefik-dynamic-config.yaml
```

- Create the following file `traefik-dynamic-config.yaml` and add the following YAML config if you are using Cloudflare
- Otherwise refer to these configuration examples for the Traefik Helm Chart for certificates and more [here](https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md)

```yaml
---
# Kubernetes secret containing the Cloudflare api token
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare
  namespace: infrastructure
type: Opaque
stringData:
  api-token: "<add your cloudflare api token>"
---
# Certificate configuration and renewal structure stored in cert-manager
# !!!!WARNING!!!! If you want to use cert-manager you need have this installed before you initiate the certifcate configuration
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cloudflare
  namespace: infrastructure
spec:
  acme:
    email: <add your email address>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: cloudflare-key
    solvers:
      - dns01:
          cloudflare:
            email: sachajw@gmail.com
            apiTokenSecretRef:
              name: cloudflare
              key: api-token
```

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: traefik
  namespace: infrastructure
spec:
  interval: 60m
  releaseName: traefik
  chart:
    spec:
      chart: traefik
      version: 30.0.0
      sourceRef:
        kind: HelmRepository
        name: traefik
      interval: 10m
  values:
    # Default values for Traefikk
    # This is a YAML-formatted file.
    # Declare variables to be passed into templates
    image:
      # -- Traefik image host registry
      registry: docker.io
      # -- Traefik image repository
      repository: traefik
      # -- defaults to appVersion
      tag:
      # -- Traefik image pull policy
      pullPolicy: IfNotPresent

    # -- Add additional label to all resources
    commonLabels: {}

    deployment:
      # -- Enable deployment
      enabled: true
      # -- Deployment or DaemonSet
      kind: Deployment
      # -- Number of pods of the deployment (only applies when kind == Deployment)
      replicas: 3
      # -- Number of old history to retain to allow rollback (If not set, default Kubernetes value is set to 10)
      # revisionHistoryLimit: 1
      # -- Amount of time (in seconds) before Kubernetes will send the SIGKILL signal if Traefik does not shut down
      terminationGracePeriodSeconds: 60
      # -- The minimum number of seconds Traefik needs to be up and running before the DaemonSet/Deployment controller considers it available
      minReadySeconds: 0
      ## Override the liveness/readiness port. This is useful to integrate traefik
      ## with an external Load Balancer that performs healthchecks.
      ## Default: ports.traefik.port
      # healthchecksPort: 9000
      ## Override the liveness/readiness host. Useful for getting ping to respond on non-default entryPoint.
      ## Default: ports.traefik.hostIP if set, otherwise Pod IP
      # healthchecksHost: localhost
      ## Override the liveness/readiness scheme. Useful for getting ping to
      ## respond on websecure entryPoint.
      # healthchecksScheme: HTTPS
      ## Override the readiness path.
      ## Default: /ping
      # readinessPath: /ping
      # Override the liveness path.
      # Default: /ping
      # livenessPath: /ping
      # -- Additional deployment annotations (e.g. for jaeger-operator sidecar injection)
      annotations: {}
      # -- Additional deployment labels (e.g. for filtering deployment by custom labels)
      labels: {}
      # -- Additional pod annotations (e.g. for mesh injection or prometheus scraping)
      # It supports templating. One can set it with values like traefik/name: '{{ template "traefik.name" . }}'
      podAnnotations: {}
      # -- Additional Pod labels (e.g. for filtering Pod by custom labels)
      podLabels: {}
      # -- Additional containers (e.g. for metric offloading sidecars)
      additionalContainers: []
      # https://docs.datadoghq.com/developers/dogstatsd/unix_socket/?tab=host
      # - name: socat-proxy
      #   image: alpine/socat:1.0.5
      #   args: ["-s", "-u", "udp-recv:8125", "unix-sendto:/socket/socket"]
      #   volumeMounts:
      #     - name: dsdsocket
      #       mountPath: /socket
      # -- Additional volumes available for use with initContainers and additionalContainers
      additionalVolumes: []
      # - name: dsdsocket
      #   hostPath:
      #     path: /var/run/statsd-exporter
      # -- Additional initContainers (e.g. for setting file permission as shown below)
      initContainers: []
      # The "volume-permissions" init container is required if you run into permission issues.
      # Related issue: https://github.com/traefik/traefik-helm-chart/issues/396
      # - name: volume-permissions
      #   image: busybox:latest
      #   command: ["sh", "-c", "touch /data/acme.json; chmod -v 600 /data/acme.json"]
      #   volumeMounts:
      #     - name: data
      #       mountPath: /data
      # -- Use process namespace sharing
      shareProcessNamespace: false
      # -- Custom pod DNS policy. Apply if `hostNetwork: true`
      # dnsPolicy: ClusterFirstWithHostNet
      # -- Custom pod [DNS config](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#poddnsconfig-v1-core)
      dnsConfig: {}
      # -- Custom [host aliases](https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/)
      hostAliases: []
      # -- Pull secret for fetching traefik container image
      imagePullSecrets: []
      # -- Pod lifecycle actions
      lifecycle: {}
      # preStop:
      #   exec:
      #     command: ["/bin/sh", "-c", "sleep 40"]
      # postStart:
      #   httpGet:
      #     path: /ping
      #     port: 9000
      #     host: localhost
      #     scheme: HTTP
      # -- Set a runtimeClassName on pod
      runtimeClassName:

    # -- [Pod Disruption Budget](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/pod-disruption-budget-v1/)
    podDisruptionBudget:
      enabled:
      maxUnavailable:
      minAvailable:

    # -- Create a default IngressClass for Traefik
    ingressClass:
      enabled: true
      isDefaultClass: true
      # name: my-custom-class

    core:
      # -- Can be used to use globally v2 router syntax
      # See https://doc.traefik.io/traefik/v3.0/migration/v2-to-v3/#new-v3-syntax-notable-changes
      defaultRuleSyntax:

    # Traefik experimental features
    experimental:
      # -- Enable traefik experimental plugins
      plugins: {}
      # demo:
      #   moduleName: github.com/traefik/plugindemo
      #   version: v0.2.1
      kubernetesGateway:
        # -- Enable traefik experimental GatewayClass CRD
        enabled: false
        ## Routes are restricted to namespace of the gateway by default.
        ## https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.FromNamespaces
        # namespacePolicy: All
        # certificate:
        #   group: "core"
        #   kind: "Secret"
        #   name: "mysecret"
        # -- By default, Gateway would be created to the Namespace you are deploying Traefik to.
        # You may create that Gateway in another namespace, setting its name below:
        # namespace: default
        # Additional gateway annotations (e.g. for cert-manager.io/issuer)
        # annotations:
        #   cert-manager.io/issuer: letsencrypt

    ingressRoute:
      dashboard:
        # -- Create an IngressRoute for the dashboard
        enabled: true
        # -- Additional ingressRoute annotations (e.g. for kubernetes.io/ingress.class)
        annotations: {}
        # -- Additional ingressRoute labels (e.g. for filtering IngressRoute by custom labels)
        labels: {}
        # -- The router match rule used for the dashboard ingressRoute
        matchRule: Host(`<add your fqdn here>`) #PathPrefix(`/dashboard`) || PathPrefix(`/api`)
        # -- Specify the allowed entrypoints to use for the dashboard ingress route, (e.g. traefik, web, websecure).
        # By default, it's using traefik entrypoint, which is not exposed.
        # /!\ Do not expose your dashboard without any protection over the internet /!\
        entryPoints: ["web", "websecure"]
        # -- Additional ingressRoute middlewares (e.g. for authentication)
        middlewares: []
        # -- TLS options (e.g. secret containing certificate)
        tls: {}
      healthcheck:
        # -- Create an IngressRoute for the healthcheck probe
        enabled: false
        # -- Additional ingressRoute annotations (e.g. for kubernetes.io/ingress.class)
        annotations: {}
        # -- Additional ingressRoute labels (e.g. for filtering IngressRoute by custom labels)
        labels: {}
        # -- The router match rule used for the healthcheck ingressRoute
        matchRule: PathPrefix(`/ping`)
        # -- Specify the allowed entrypoints to use for the healthcheck ingress route, (e.g. traefik, web, websecure).
        # By default, it's using traefik entrypoint, which is not exposed.
        entryPoints: ["traefik"]
        # -- Additional ingressRoute middlewares (e.g. for authentication)
        middlewares: []
        # -- TLS options (e.g. secret containing certificate)
        tls: {}

    updateStrategy:
      # -- Customize updateStrategy: RollingUpdate or OnDelete
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 0
        maxSurge: 1

    readinessProbe:
      # -- The number of consecutive failures allowed before considering the probe as failed.
      failureThreshold: 1
      # -- The number of seconds to wait before starting the first probe.
      initialDelaySeconds: 2
      # -- The number of seconds to wait between consecutive probes.
      periodSeconds: 10
      # -- The minimum consecutive successes required to consider the probe successful.
      successThreshold: 1
      # -- The number of seconds to wait for a probe response before considering it as failed.
      timeoutSeconds: 2
    livenessProbe:
      # -- The number of consecutive failures allowed before considering the probe as failed.
      failureThreshold: 3
      # -- The number of seconds to wait before starting the first probe.
      initialDelaySeconds: 2
      # -- The number of seconds to wait between consecutive probes.
      periodSeconds: 10
      # -- The minimum consecutive successes required to consider the probe successful.
      successThreshold: 1
      # -- The number of seconds to wait for a probe response before considering it as failed.
      timeoutSeconds: 2

    # -- Define [Startup Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes)
    startupProbe:

    providers:
      kubernetesCRD:
        # -- Load Kubernetes IngressRoute provider
        enabled: true
        # -- Allows IngressRoute to reference resources in namespace other than theirs
        allowCrossNamespace: false
        # -- Allows to reference ExternalName services in IngressRoute
        allowExternalNameServices: false
        # -- Allows to return 503 when there is no endpoints available
        allowEmptyServices: false
        # -- When the parameter is set, only resources containing an annotation with the same value are processed. Otherwise, resources missing the annotation, having an empty value, or the value traefik are processed. It will also set required annotation on Dashboard and Healthcheck IngressRoute when enabled.
        ingressClass:
        # labelSelector: environment=production,method=traefik
        # -- Array of namespaces to watch. If left empty, Traefik watches all namespaces.
        namespaces: []
        # -- Defines whether to use Native Kubernetes load-balancing mode by default.
        nativeLBByDefault:

      kubernetesIngress:
        # -- Load Kubernetes Ingress provider
        enabled: true
        # -- Allows to reference ExternalName services in Ingress
        allowExternalNameServices: false
        # -- Allows to return 503 when there is no endpoints available
        allowEmptyServices: false
        # -- When ingressClass is set, only Ingresses containing an annotation with the same value are processed. Otherwise, Ingresses missing the annotation, having an empty value, or the value traefik are processed.
        ingressClass:
        # labelSelector: environment=production,method=traefik
        # -- Array of namespaces to watch. If left empty, Traefik watches all namespaces.
        namespaces: []
        # - "default"
        # Disable cluster IngressClass Lookup - Requires Traefik V3.
        # When combined with rbac.namespaced: true, ClusterRole will not be created and ingresses must use kubernetes.io/ingress.class annotation instead of spec.ingressClassName.
        disableIngressClassLookup: false
        # IP used for Kubernetes Ingress endpoints
        publishedService:
          enabled: false
          # Published Kubernetes Service to copy status from. Format: namespace/servicename
          # By default this Traefik service
          # pathOverride: ""
        # -- Defines whether to use Native Kubernetes load-balancing mode by default.
        nativeLBByDefault:

      file:
        # -- Create a file provider
        enabled: false
        # -- Allows Traefik to automatically watch for file changes
        watch: true
        # -- File content (YAML format, go template supported) (see https://doc.traefik.io/traefik/providers/file/)
        # content:
        providers:
          file:
            directory: /manifests/traefik-dynamic-config.yaml

    # -- Add volumes to the traefik pod. The volume name will be passed to tpl.
    # This can be used to mount a cert pair or a configmap that holds a config.toml file.
    # After the volume has been mounted, add the configs into traefik by using the `additionalArguments` list below, eg:
    # `additionalArguments:
    # - "--providers.file.filename=/config/dynamic.toml"
    # - "--ping"
    # - "--ping.entrypoint=web"`
    volumes: []
    # - name: public-cert
    #   mountPath: "/certs"
    #   type: secret
    # - name: '{{ printf "%s-configs" .Release.Name }}'
    #   mountPath: "/config"
    #   type: configMap

    # -- Additional volumeMounts to add to the Traefik container
    additionalVolumeMounts: []
    # -- For instance when using a logshipper for access logs
    # - name: traefik-logs
    #   mountPath: /var/log/traefik

    logs:
      general:
        # -- Set [logs format](https://doc.traefik.io/traefik/observability/logs/#format)
        # @default common
        format:
        # By default, the level is set to ERROR.
        # -- Alternative logging levels are DEBUG, PANIC, FATAL, ERROR, WARN, and INFO.
        level: INFO
      access:
        # -- To enable access logs
        enabled: false
        # -- Set [access log format](https://doc.traefik.io/traefik/observability/access-logs/#format)
        format:
        # filePath: "/var/log/traefik/access.log
        # -- Set [bufferingSize](https://doc.traefik.io/traefik/observability/access-logs/#bufferingsize)
        bufferingSize:
        # -- Set [filtering](https://docs.traefik.io/observability/access-logs/#filtering)
        filters: {}
        # statuscodes: "200,300-302"
        # retryattempts: true
        # minduration: 10ms
        # -- Enables accessLogs for internal resources. Default: false.
        addInternals:
        fields:
          general:
            # -- Available modes: keep, drop, redact.
            defaultmode: keep
            # -- Names of the fields to limit.
            names: {}
            ## Examples:
            # ClientUsername: drop
          # -- [Limit logged fields or headers](https://doc.traefik.io/traefik/observability/access-logs/#limiting-the-fieldsincluding-headers)
          headers:
            # -- Available modes: keep, drop, redact.
            defaultmode: drop
            names: {}

    metrics:
      ## -- Enable metrics for internal resources. Default: false
      addInternals:

      ## -- Prometheus is enabled by default.
      ## -- It can be disabled by setting "prometheus: null"
      prometheus:
        # -- Entry point used to expose metrics.
        entryPoint: metrics
        ## Enable metrics on entry points. Default=true
        # addEntryPointsLabels: false
        ## Enable metrics on routers. Default=false
        # addRoutersLabels: true
        ## Enable metrics on services. Default=true
        # addServicesLabels: false
        ## Buckets for latency metrics. Default="0.1,0.3,1.2,5.0"
        # buckets: "0.5,1.0,2.5"
        ## When manualRouting is true, it disables the default internal router in
        ## order to allow creating a custom router for prometheus@internal service.
        # manualRouting: true
        service:
          # -- Create a dedicated metrics service to use with ServiceMonitor
          enabled:
          labels:
          annotations:
        # -- When set to true, it won't check if Prometheus Operator CRDs are deployed
        disableAPICheck:
        serviceMonitor:
          # -- Enable optional CR for Prometheus Operator. See EXAMPLES.md for more details.
          enabled: false
          metricRelabelings:
          relabelings:
          jobLabel:
          interval:
          honorLabels:
          scrapeTimeout:
          honorTimestamps:
          enableHttp2:
          followRedirects:
          additionalLabels:
          namespace:
          namespaceSelector:
        prometheusRule:
          # -- Enable optional CR for Prometheus Operator. See EXAMPLES.md for more details.
          enabled: false
          additionalLabels:
          namespace:

      #  datadog:
      #    ## Address instructs exporter to send metrics to datadog-agent at this address.
      #    address: "127.0.0.1:8125"
      #    ## The interval used by the exporter to push metrics to datadog-agent. Default=10s
      #    # pushInterval: 30s
      #    ## The prefix to use for metrics collection. Default="traefik"
      #    # prefix: traefik
      #    ## Enable metrics on entry points. Default=true
      #    # addEntryPointsLabels: false
      #    ## Enable metrics on routers. Default=false
      #    # addRoutersLabels: true
      #    ## Enable metrics on services. Default=true
      #    # addServicesLabels: false
      #  influxdb2:
      #    ## Address instructs exporter to send metrics to influxdb v2 at this address.
      #    address: localhost:8086
      #    ## Token with which to connect to InfluxDB v2.
      #    token: xxx
      #    ## Organisation where metrics will be stored.
      #    org: ""
      #    ## Bucket where metrics will be stored.
      #    bucket: ""
      #    ## The interval used by the exporter to push metrics to influxdb. Default=10s
      #    # pushInterval: 30s
      #    ## Additional labels (influxdb tags) on all metrics.
      #    # additionalLabels:
      #    #   env: production
      #    #   foo: bar
      #    ## Enable metrics on entry points. Default=true
      #    # addEntryPointsLabels: false
      #    ## Enable metrics on routers. Default=false
      #    # addRoutersLabels: true
      #    ## Enable metrics on services. Default=true
      #    # addServicesLabels: false
      #  statsd:
      #    ## Address instructs exporter to send metrics to statsd at this address.
      #    address: localhost:8125
      #    ## The interval used by the exporter to push metrics to influxdb. Default=10s
      #    # pushInterval: 30s
      #    ## The prefix to use for metrics collection. Default="traefik"
      #    # prefix: traefik
      #    ## Enable metrics on entry points. Default=true
      #    # addEntryPointsLabels: false
      #    ## Enable metrics on routers. Default=false
      #    # addRoutersLabels: true
      #    ## Enable metrics on services. Default=true
      #    # addServicesLabels: false
      otlp:
        # -- Set to true in order to enable the OpenTelemetry metrics
        enabled: false
        # -- Enable metrics on entry points. Default: true
        addEntryPointsLabels:
        # -- Enable metrics on routers. Default: false
        addRoutersLabels:
        # -- Enable metrics on services. Default: true
        addServicesLabels:
        # -- Explicit boundaries for Histogram data points. Default: [.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
        explicitBoundaries:
        # -- Interval at which metrics are sent to the OpenTelemetry Collector. Default: 10s
        pushInterval:
        http:
          # -- Set to true in order to send metrics to the OpenTelemetry Collector using HTTP.
          enabled: false
          # -- Format: <scheme>://<host>:<port><path>. Default: http://localhost:4318/v1/metrics
          endpoint:
          # -- Additional headers sent with metrics by the reporter to the OpenTelemetry Collector.
          headers:
          ## Defines the TLS configuration used by the reporter to send metrics to the OpenTelemetry Collector.
          tls:
            # -- The path to the certificate authority, it defaults to the system bundle.
            ca:
            # -- The path to the public certificate. When using this option, setting the key option is required.
            cert:
            # -- The path to the private key. When using this option, setting the cert option is required.
            key:
            # -- When set to true, the TLS connection accepts any certificate presented by the server regardless of the hostnames it covers.
            insecureSkipVerify:
        grpc:
          # -- Set to true in order to send metrics to the OpenTelemetry Collector using gRPC
          enabled: false
          # -- Format: <scheme>://<host>:<port><path>. Default: http://localhost:4318/v1/metrics
          endpoint:
          # -- Allows reporter to send metrics to the OpenTelemetry Collector without using a secured protocol.
          insecure:
          ## Defines the TLS configuration used by the reporter to send metrics to the OpenTelemetry Collector.
          tls:
            # -- The path to the certificate authority, it defaults to the system bundle.
            ca:
            # -- The path to the public certificate. When using this option, setting the key option is required.
            cert:
            # -- The path to the private key. When using this option, setting the cert option is required.
            key:
            # -- When set to true, the TLS connection accepts any certificate presented by the server regardless of the hostnames it covers.
            insecureSkipVerify:

    ## Tracing
    # -- https://doc.traefik.io/traefik/observability/tracing/overview/
    tracing:
      # -- Enables tracing for internal resources. Default: false.
      addInternals:
      otlp:
        # -- See https://doc.traefik.io/traefik/v3.0/observability/tracing/opentelemetry/
        enabled: false
        http:
          # -- Set to true in order to send metrics to the OpenTelemetry Collector using HTTP.
          enabled: false
          # -- Format: <scheme>://<host>:<port><path>. Default: http://localhost:4318/v1/metrics
          endpoint:
          # -- Additional headers sent with metrics by the reporter to the OpenTelemetry Collector.
          headers:
          ## Defines the TLS configuration used by the reporter to send metrics to the OpenTelemetry Collector.
          tls:
            # -- The path to the certificate authority, it defaults to the system bundle.
            ca:
            # -- The path to the public certificate. When using this option, setting the key option is required.
            cert:
            # -- The path to the private key. When using this option, setting the cert option is required.
            key:
            # -- When set to true, the TLS connection accepts any certificate presented by the server regardless of the hostnames it covers.
            insecureSkipVerify:
        grpc:
          # -- Set to true in order to send metrics to the OpenTelemetry Collector using gRPC
          enabled: false
          # -- Format: <scheme>://<host>:<port><path>. Default: http://localhost:4318/v1/metrics
          endpoint:
          # -- Allows reporter to send metrics to the OpenTelemetry Collector without using a secured protocol.
          insecure:
          ## Defines the TLS configuration used by the reporter to send metrics to the OpenTelemetry Collector.
          tls:
            # -- The path to the certificate authority, it defaults to the system bundle.
            ca:
            # -- The path to the public certificate. When using this option, setting the key option is required.
            cert:
            # -- The path to the private key. When using this option, setting the cert option is required.
            key:
            # -- When set to true, the TLS connection accepts any certificate presented by the server regardless of the hostnames it covers.
            insecureSkipVerify:

    # -- Global command arguments to be passed to all traefik's pods
    globalArguments:
      - "--global.checknewversion"
      # - "--global.sendanonymoususage"

    # -- Additional arguments to be passed at Traefik's binary
    # See [CLI Reference](https://docs.traefik.io/reference/static-configuration/cli/)
    # Use curly braces to pass values: `helm install --set="additionalArguments={--providers.kubernetesingress.ingressclass=traefik-internal,--log.level=DEBUG}"`
    additionalArguments: []
    #  - "--providers.kubernetesingress.ingressclass=traefik-internal"
    #  - "--log.level=DEBUG"

    # -- Environment variables to be passed to Traefik's binary
    # @default -- See _values.yaml_
    env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace

    # -- Environment variables to be passed to Traefik's binary from configMaps or secrets
    envFrom: []

    ports:
      traefik:
        port: 9000
        # -- Use hostPort if set.
        # hostPort: 9000
        #
        # -- Use hostIP if set. If not set, Kubernetes will default to 0.0.0.0, which
        # means it's listening on all your interfaces and all your IPs. You may want
        # to set this value if you need traefik to listen on specific interface
        # only.
        # hostIP: 192.168.100.10

        # Defines whether the port is exposed if service.type is LoadBalancer or
        # NodePort.
        #
        # -- You SHOULD NOT expose the traefik port on production deployments.
        # If you want to access it from outside your cluster,
        # use `kubectl port-forward` or create a secure ingress
        expose:
          default: false
        # -- The exposed port for this service
        exposedPort: 9000
        # -- The port protocol (TCP/UDP)
        protocol: TCP
      web:
        ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
        # asDefault: true
        port: 8000
        # hostPort: 8000
        # containerPort: 8000
        expose:
          default: true
        exposedPort: 80
        ## -- Different target traefik port on the cluster, useful for IP type LB
        # targetPort: 80
        # The port protocol (TCP/UDP)
        protocol: TCP
        # -- Use nodeport if set. This is useful if you have configured Traefik in a
        # LoadBalancer.
        # nodePort: 32080
        # Port Redirections
        # Added in 2.2, you can make permanent redirects via entrypoints.
        # https://docs.traefik.io/routing/entrypoints/#redirection
        # redirectTo:
        #   port: websecure
        #   (Optional)
        #   priority: 10
        #   permanent: true
        #
        # -- Trust forwarded headers information (X-Forwarded-*).
        # forwardedHeaders:
        #   trustedIPs: []
        #   insecure: false
        #
        # -- Enable the Proxy Protocol header parsing for the entry point
        # proxyProtocol:
        #   trustedIPs: []
        #   insecure: false
        #
        # -- Set transport settings for the entrypoint; see also
        # https://doc.traefik.io/traefik/routing/entrypoints/#transport
        transport:
          respondingTimeouts:
            readTimeout:
            writeTimeout:
            idleTimeout:
          lifeCycle:
            requestAcceptGraceTimeout:
            graceTimeOut:
          keepAliveMaxRequests:
          keepAliveMaxTime:
      websecure:
        ## -- Enable this entrypoint as a default entrypoint. When a service doesn't explicitly set an entrypoint it will only use this entrypoint.
        # asDefault: true
        port: 8443
        # hostPort: 8443
        # containerPort: 8443
        expose:
          default: true
        exposedPort: 443
        ## -- Different target traefik port on the cluster, useful for IP type LB
        # targetPort: 80
        ## -- The port protocol (TCP/UDP)
        protocol: TCP
        # nodePort: 32443
        ## -- Specify an application protocol. This may be used as a hint for a Layer 7 load balancer.
        # appProtocol: https
        #
        ## -- Enable HTTP/3 on the entrypoint
        ## Enabling it will also enable http3 experimental feature
        ## https://doc.traefik.io/traefik/routing/entrypoints/#http3
        ## There are known limitations when trying to listen on same ports for
        ## TCP & UDP (Http3). There is a workaround in this chart using dual Service.
        ## https://github.com/kubernetes/kubernetes/issues/47249#issuecomment-587960741
        http3:
          enabled: true
        # advertisedPort: 4443
        #
        # -- Trust forwarded headers information (X-Forwarded-*).
        # forwardedHeaders:
        #   trustedIPs: []
        #   insecure: false
        #
        # -- Enable the Proxy Protocol header parsing for the entry point
        # proxyProtocol:
        #   trustedIPs: []
        #   insecure: false
        #
        # -- Set transport settings for the entrypoint; see also
        # https://doc.traefik.io/traefik/routing/entrypoints/#transport
        transport:
          respondingTimeouts:
            readTimeout:
            writeTimeout:
            idleTimeout:
          lifeCycle:
            requestAcceptGraceTimeout:
            graceTimeOut:
          keepAliveMaxRequests:
          keepAliveMaxTime:
        #
        ## Set TLS at the entrypoint
        ## https://doc.traefik.io/traefik/routing/entrypoints/#tls
        tls:
          enabled: true
          # this is the name of a TLSOption definition
          options: ""
          certResolver: ""
          domains: []
          # - main: example.com
          #   sans:
          #     - foo.example.com
          #     - bar.example.com
        #
        # -- One can apply Middlewares on an entrypoint
        # https://doc.traefik.io/traefik/middlewares/overview/
        # https://doc.traefik.io/traefik/routing/entrypoints/#middlewares
        # -- /!\ It introduces here a link between your static configuration and your dynamic configuration /!\
        # It follows the provider naming convention: https://doc.traefik.io/traefik/providers/overview/#provider-namespace
        # middlewares:
        #   - namespace-name1@kubernetescrd
        #   - namespace-name2@kubernetescrd
        middlewares: []
      metrics:
        # -- When using hostNetwork, use another port to avoid conflict with node exporter:
        # https://github.com/prometheus/prometheus/wiki/Default-port-allocations
        port: 9100
        # hostPort: 9100
        # Defines whether the port is exposed if service.type is LoadBalancer or
        # NodePort.
        #
        # -- You may not want to expose the metrics port on production deployments.
        # If you want to access it from outside your cluster,
        # use `kubectl port-forward` or create a secure ingress
        expose:
          default: false
        # -- The exposed port for this service
        exposedPort: 9100
        # -- The port protocol (TCP/UDP)
        protocol: TCP

    # -- TLS Options are created as [TLSOption CRDs](https://doc.traefik.io/traefik/https/tls/#tls-options)
    # When using `labelSelector`, you'll need to set labels on tlsOption accordingly.
    # See EXAMPLE.md for details.
    tlsOptions: {}

    # -- TLS Store are created as [TLSStore CRDs](https://doc.traefik.io/traefik/https/tls/#default-certificate). This is useful if you want to set a default certificate. See EXAMPLE.md for details.
    tlsStore: {}

    service:
      enabled: true
      ## -- Single service is using `MixedProtocolLBService` feature gate.
      ## -- When set to false, it will create two Service, one for TCP and one for UDP.
      single: true
      type: LoadBalancer
      # -- Additional annotations applied to both TCP and UDP services (e.g. for cloud provider specific config)
      annotations: {}
      # -- Additional annotations for TCP service only
      annotationsTCP: {}
      # -- Additional annotations for UDP service only
      annotationsUDP: {}
      # -- Additional service labels (e.g. for filtering Service by custom labels)
      labels: {}
      # -- Additional entries here will be added to the service spec.
      # -- Cannot contain type, selector or ports entries.
      spec: {}
      # externalTrafficPolicy: Cluster
      # loadBalancerIP: "1.2.3.4"
      # clusterIP: "2.3.4.5"
      loadBalancerSourceRanges: []
      # - 192.168.0.1/32
      # - 172.16.0.0/16
      ## -- Class of the load balancer implementation
      # loadBalancerClass: service.k8s.aws/nlb
      externalIPs: []
      # - 1.2.3.4
      ## One of SingleStack, PreferDualStack, or RequireDualStack.
      # ipFamilyPolicy: SingleStack
      ## List of IP families (e.g. IPv4 and/or IPv6).
      ## ref: https://kubernetes.io/docs/concepts/services-networking/dual-stack/#services
      # ipFamilies:
      #   - IPv4
      #   - IPv6
      ##
      additionalServices: {}
      ## -- An additional and optional internal Service.
      ## Same parameters as external Service
      # internal:
      #   type: ClusterIP
      #   # labels: {}
      #   # annotations: {}
      #   # spec: {}
      #   # loadBalancerSourceRanges: []
      #   # externalIPs: []
      #   # ipFamilies: [ "IPv4","IPv6" ]

    autoscaling:
      # -- Create HorizontalPodAutoscaler object.
      # See EXAMPLES.md for more details.
      enabled: false

    persistence:
      # -- Enable persistence using Persistent Volume Claims
      # ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
      # It can be used to store TLS certificates, see `storage` in certResolvers
      enabled: false
      name: data
      #  existingClaim: ""
      accessMode: ReadWriteOnce
      size: 128Mi
      storageClass: "-"
      # volumeName: ""
      path: /data
      annotations: {}
      # -- Only mount a subpath of the Volume into the pod
      # subPath: ""

    # -- Certificates resolvers configuration.
    # Ref: https://doc.traefik.io/traefik/https/acme/#certificate-resolvers
    # See EXAMPLES.md for more details.
    certResolvers: {}

    # -- If hostNetwork is true, runs traefik in the host network namespace
    # To prevent unschedulabel pods due to port collisions, if hostNetwork=true
    # and replicas>1, a pod anti-affinity is recommended and will be set if the
    # affinity is left as default.
    hostNetwork: false

    # -- Whether Role Based Access Control objects like roles and rolebindings should be created
    rbac:
      enabled: true
      # If set to false, installs ClusterRole and ClusterRoleBinding so Traefik can be used across namespaces.
      # If set to true, installs Role and RoleBinding instead of ClusterRole/ClusterRoleBinding. Providers will only watch target namespace.
      # When combined with providers.kubernetesIngress.disableIngressClassLookup: true and Traefik V3, ClusterRole to watch IngressClass is also disabled.
      namespaced: false
      # Enable user-facing roles
      # https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles
      # aggregateTo: [ "admin" ]
      # List of Kubernetes secrets that are accessible for Traefik. If empty, then access is granted to every secret.
      secretResourceNames: []

    # -- Enable to create a PodSecurityPolicy and assign it to the Service Account via RoleBinding or ClusterRoleBinding
    podSecurityPolicy:
      enabled: false

    # -- The service account the pods will use to interact with the Kubernetes API
    serviceAccount:
      # If set, an existing service account is used
      # If not set, a service account is created automatically using the fullname template
      name: ""

    # -- Additional serviceAccount annotations (e.g. for oidc authentication)
    serviceAccountAnnotations: {}

    # -- [Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for `traefik` container.
    resources: {}

    # -- This example pod anti-affinity forces the scheduler to put traefik pods
    # -- on nodes where no other traefik pods are scheduled.
    # It should be used when hostNetwork: true to prevent port conflicts
    affinity: {}
    #  podAntiAffinity:
    #    requiredDuringSchedulingIgnoredDuringExecution:
    #      - labelSelector:
    #          matchLabels:
    #            app.kubernetes.io/name: '{{ template "traefik.name" . }}'
    #            app.kubernetes.io/instance: '{{ .Release.Name }}-{{ .Release.Namespace }}'
    #        topologyKey: kubernetes.io/hostname

    # -- nodeSelector is the simplest recommended form of node selection constraint.
    nodeSelector: {}
    # -- Tolerations allow the scheduler to schedule pods with matching taints.
    tolerations: []
    # -- You can use topology spread constraints to control
    # how Pods are spread across your cluster among failure-domains.
    topologySpreadConstraints: []
    # This example topologySpreadConstraints forces the scheduler to put traefik pods
    # on nodes where no other traefik pods are scheduled.
    #  - labelSelector:
    #      matchLabels:
    #        app: '{{ template "traefik.name" . }}'
    #    maxSkew: 1
    #    topologyKey: kubernetes.io/hostname
    #    whenUnsatisfiable: DoNotSchedule

    # -- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
    priorityClassName: ""

    # -- [SecurityContext](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context-1)
    # @default -- See _values.yaml_
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      readOnlyRootFilesystem: true

    # -- [Pod Security Context](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context)
    # @default -- See _values.yaml_
    podSecurityContext:
      runAsGroup: 65532
      runAsNonRoot: true
      runAsUser: 65532

    #
    # -- Extra objects to deploy (value evaluated as a template)
    #
    # In some cases, it can avoid the need for additional, extended or adhoc deployments.
    # See #595 for more details and traefik/tests/values/extra.yaml for example.
    extraObjects: []

    # -- This field override the default Release Namespace for Helm.
    # It will not affect optional CRDs such as `ServiceMonitor` and `PrometheusRules`
    namespaceOverride:

    ## -- This field override the default app.kubernetes.io/instance label for all Objects.
    instanceLabelOverride:

    # Traefik Hub configuration. See https://doc.traefik.io/traefik-hub/
    hub:
      # -- Name of `Secret` with key 'token' set to a valid license token.
      # It enables API Gateway.
      token:
      apimanagement:
        # -- Set to true in order to enable API Management. Requires a valid license token.
        enabled:
        admission:
          # -- WebHook admission server listen address. Default: "0.0.0.0:9943".
          listenAddr:
          # -- Certificate of the WebHook admission server. Default: "hub-agent-cert".
          secretName:

      ratelimit:
        redis:
          # -- Enable Redis Cluster. Default: true.
          cluster:
          # -- Database used to store information. Default: "0".
          database:
          # -- Endpoints of the Redis instances to connect to. Default: "".
          endpoints:
          # -- The username to use when connecting to Redis endpoints. Default: "".
          username:
          # -- The password to use when connecting to Redis endpoints. Default: "".
          password:
          sentinel:
            # -- Name of the set of main nodes to use for main selection. Required when using Sentinel. Default: "".
            masterset:
            # -- Username to use for sentinel authentication (can be different from endpoint username). Default: "".
            username:
            # -- Password to use for sentinel authentication (can be different from endpoint password). Default: "".
            password:
          # -- Timeout applied on connection with redis. Default: "0s".
          timeout:
          tls:
            # -- Path to the certificate authority used for the secured connection.
            ca:
            # -- Path to the public certificate used for the secure connection.
            cert:
            # -- Path to the private key used for the secure connection.
            key:
            # -- When insecureSkipVerify is set to true, the TLS connection accepts any certificate presented by the server. Default: false.
            insecureSkipVerify:
      # Enable export of errors logs to the platform. Default: true.
      sendlogs:
```

- Lets git it

```shell
git add .
git commit -m "k8s infra metallb"
git push
```

#### Fluxcd is doing the following under the hood

- Helm repo add

```shell
helm repo add traefik https://traefik.github.io/charts
```

- Kubectl create the Traefik namespace

```shell
kubectl create ns traefik-v2
```

- Kubectl switch to the traefik-v2 namespace

```shell
kubectl config set-context --current --namespace=traefik-v2
```

- Helm repo update

```shell
helm repo update
```

- Helm install Traefik

```shell
helm install traefik traefik/traefik --namespace=traefik-v2
```


- Kubectl show me the pods for Traefik

```shell
kubectl get pods
```

![traefik pods](images/how-to-bake-an-ortelius-pi/part03/06-traefik-pods.png)

- Using GitHub fork the [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)
- Clone the Helm Chart to your local machine and enable the Traefik `dashboard, kubernetesCRD and kubernetesIngress` in `values.yaml` and don't forget to save
- `FYI` they might already be enabled

```yaml
## Create an IngressRoute for the dashboard
ingressRoute:
  dashboard:
    # -- Create an IngressRoute for the dashboard
    enabled: true
```

```yaml
providers:
  kubernetesCRD:
    # -- Load Kubernetes IngressRoute provider
    enabled: true
```

```yaml
  kubernetesIngress:
    # -- Load Kubernetes Ingress provider
    enabled: true
```

- Because Traefik is deployed with Helm we will use Helm to update our deployment from `values.yaml`

```shell
helm upgrade traefik traefik/traefik --values values.yaml
```

- Now we need to deploy an `ingress route` which forms part of the [CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) that were installed with Traefik
- CRDs are custom resources created in our Kubernetes cluster that add additional magic
- Kubectl show me all CRDs for Traefik

```shell
kubectl get crds | grep traefik
```

![traefik pod](images/how-to-bake-an-ortelius-pi/part03/07-traefik-crds.png)

- Create a file called `dashboard.yaml` and apply the following logic with `kubectl apply -f dashboard.yaml`
- You will need a DNS record created either on your DNS server or in localhosts file to access the dashboard
- Edit localhosts on Linux and Mac here with sudo rights `sudo vi /etc/hosts` by adding `your private ip and traefik.yourdomain.your tld`
- Edit Windows localhosts file here as administrator `windows\System32\drivers\etc\hosts` by adding `your private ip and traefik.yourdomain.your tld`
- [TLD = Top Level Domain](https://en.wikipedia.org/wiki/Top-level_domain)

```yaml
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

```shell
kubectl get ingressroutes.traefik.io
```

![traefik pod](images/how-to-bake-an-ortelius-pi/part03/08-traefik-ingressroute-dashboard.png)

- Kubectl show me that the Traefik service has claimed our MetalLB single IP address

```shell
kubectl get svc
```

![traefik service](images/how-to-bake-an-ortelius-pi/part03/09-traefik-service.png)

- Here is a view of the services for all namespaces

```shell
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
