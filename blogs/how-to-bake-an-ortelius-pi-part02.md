## How to bake an Ortelius Pi Part 2 | The Preparation

#### Introduction

In [part 1](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-1-the-hardware/) of this series we prepped our Raspberry Pi 4's for the installation of Ubuntu 22.04.4 LTS. In part 2 we will prepare our three Pi's for NFS (Network File System) storage with a Synology NAS and install MicroK8s [MicroK8s](https://microk8s.io/).

#### OS Prep
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
- Install Microk8s on each Pi ```sudo snap install microk8s --classic`` - This install the latest version of Microk8s








#### Conclusion

By this stage you should have three Pi 4 B's running with Ubuntu 22.04.4 LTS and MicroK8s. Stay tuned for part 3 where we will dive into optimising USB flash sticks for the best performance and stability and the installation of MicroK8s.

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
