## How to bake an Ortelius Pi Part 2 | The Preparation

#### Introduction

In [part one](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-1-the-hardware/) of this series we installed Ubuntu 22.04.4 LTS on our Raspberry Pi's. In part two we will prepare our three Pi's for DNS, NFS (Network File System) storage with a Synology NAS and install [MicroK8s](https://microk8s.io/).

#### DNS

For DNS I use [NextDNS](https://nextdns.io/) but this is not just DNS its complete protection for all your devices (mobiles, tablets, routers, home internet, NAS devices, laptops and desktops) including your Pi MicroK8s nodes. It will use `127.0.0.1:53` to resolve your local IPs but to do that we need to do some configuration by logging into the NextDNS portal and installing the cli.

If you are using NextDNS do the following
- Go [here](https://github.com/nextdns/nextdns/wiki) - You will find usage instruction here too
- Install the cli on each Pi and on your NAS
```
sh -c 'sh -c "$(curl -sL https://nextdns.io/install)"'
```
- Run `sudo nextdns config` to view your config
- Run `sudo nextdns config edit` to edit for each Pi and NAS and configure like this
```
debug false
cache-size 10MB
max-ttl 5s
mdns all
cache-max-age 0s
use-hosts true
auto-activate true
listen localhost:53
profile <your profile id goes here>
log-queries false
report-client-info true
discovery-dns
hardened-privacy false
timeout 5s
setup-router false
control /var/run/nextdns.sock
detect-captive-portals false
bogus-priv true
max-inflight-requests 256
```
- Run `sudo nextdns restart` to restart the service
- Run `sudo nextdns status` to check the service status
- Then in your NextDNS portal go to `Settings`

![nextdns settings](images/how-to-bake-an-ortelius-pi/part02/15-dns-settings.png)

- Then scroll down to `Rewrites`

![nextdns rewrites](images/how-to-bake-an-ortelius-pi/part02/16-dns-rewrites.png)

- And add your private network DNS records for example

![nextdns records](images/how-to-bake-an-ortelius-pi/part02/17-dns-records.png)

- NextDNS will instantly auto refresh all your NextDNS agents with any changes

#### OS Prep
- Install Kubectl [here](https://kubernetes.io/docs/tasks/tools/) on your local machine
- Install Helm [here](https://helm.sh/) on your local machine
- SSH into each Pi like this `ssh <your username>@<your ip address>` and your password
- Update all packages to the latest with `sudo apt update -y && sudo apt upgrade -y` then go and make coffee

#### NFS Prep

I am using a Synology DS413j with DSM 6.2.4-25556 Update 7 so the following steps will be inline with my Synology
- Install `sudo apt install nfs-common -y` for each Pi

#### Enable NFS on the Synology
- Login to the Synology and go to `File Services`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/01-syno-file-services-icon.png)

- On the `SMB/AFP/NFS` tab and scroll until you see `NFS` and `enable NFS and enable NFSv4 support`

![synology nfs services](images/how-to-bake-an-ortelius-pi/part02/02-syno-nfs-enable-tab.png)

![synology nfs services](images/how-to-bake-an-ortelius-pi/part02/03-syno-nfs-enable.png)

#### Configure Shared Folder
- Go to `File Sharing`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/04-syno-file-sharing-icon.png)

- Click `Create`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/05-syno-file-sharing-create.png)

- Create a name for your folder share, I used `Pi8s`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/06-syno-file-sharing-next.png)

- Skip encryption
- Apply your config

![synology file services](images/how-to-bake-an-ortelius-pi/part02/07-syno-file-sharing-confirm.png)

- Right click your newly created `Shared Folder` and select `Edit`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/08-syno-file-share-edit.png)

- Select `Permissions` tab

![synology file services](images/how-to-bake-an-ortelius-pi/part02/09-syno-file-share-permissions.png)

- Select `Local users`drop down and give the  `admin` `Read/Write` permissions by checking the box

![synology file services](images/how-to-bake-an-ortelius-pi/part02/10-syno-file-share-local-users.png)

![synology file services](images/how-to-bake-an-ortelius-pi/part02/11-syno-file-share-admin.png)

- Select `NFS Permissions` and then `Create`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/12-syno-file-share-nfs-permissions.png)

![synology file services](images/how-to-bake-an-ortelius-pi/part02/13-syno-file-share-nfs-create.png)

- Configure like this then click `OK`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/14-syno-file-share-nfs-config.png)

- Congrats you just configured the Synology for NFS!

#### MicroK8s Prep
- [MicroK8s docs](https://microk8s.io/docs)
- Configure Pi BIOS `sudo vi /boot/firmware/cmdline.txt` and add the following `cgroup_enable=memory cgroup_memory=1`
- Below is the config from my Pi
```
cgroup_enable=memory cgroup_memory=1 console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait fixrtc quiet splash
```
- Install Kernel Modules `sudo apt install linux-modules-extra-raspi`
- Referenced from [here](https://microk8s.io/docs/install-raspberry-pi)
- Install Microk8s on each Pi
```
sudo snap install microk8s --classic
```
- This installs the latest version of Microk8s

#### MicroK8s create highly available 3 node cluster
- Choose a Pi to start the process, I used `pi01`
- SSH onto `pi01` and run this command on `pi01`
```
sudo microk8s add-node
```
- You will need to run this 3 times on the same node to generate a unique key for each node you wish to join
- This will return some joining instructions which should be executed on the MicroK8s instance that you wish to join to the cluster `(NOT THE NODE YOU RAN add-node FROM)`
```
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05 --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05
microk8s join 10.23.209.1:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05
microk8s join 172.17.0.1:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05
```
- Referenced from [here](https://microk8s.io/docs/clustering)
- On the same Pi run `sudo microk8s config`
- This will return config you will need to access your Microk8s cluster
- On your computer you will need to configure Kubectl
- My Kubectl configuration is here on my Mac `/home/.kube/config`
```
- cluster:
    certificate-authority-data: <your certificate authority data goes here>
    server: https://<your local network IP for your Pi goes here>:16443
  name: microk8s-cluster
- context:
    cluster: microk8s-cluster
    namespace: default
    user: <your user goes here>
  name: microk8s
users:
- name: <your user goes here>
  user:
    client-certificate-data: <your client certificate data goes here>
```

#### Conclusion

By this stage you should have three Pi's each running with NFS and MicroK8s installed. Stay tuned for part 3 where we will install the NSF `csi-driver-nfs` for Kubernetes

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
