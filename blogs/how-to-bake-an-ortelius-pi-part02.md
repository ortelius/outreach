## How to bake an Ortelius Pi Part 2 | The Preparation

#### Introduction

In the first [part](https://ortelius.io/blog/2024/03/27/how-to-bake-an-ortelius-pi-part-1-the-hardware/) of this series we prepped our Raspberry Pi 4's for the installation of Ubuntu 22.04.4 LTS. In part 2 we will prepare our three Pi's to install MicroK8s [MicroK8s](https://microk8s.io/) and NFS (Network File System) storage with our Synology.

#### OS Prep
- SSH into your Pi like in this example `ssh admin@192.168.0.1`
- Update all packages to the latest with `sudo apt update -y && sudo apt upgrade -y` then go and make coffee
- then `sudo vi /boot/firmware/cmdline.txt`
- Add the following `cgroup_enable=memory cgroup_memory=1`
- Kernel Modules installation `sudo apt install linux-modules-extra-raspi`
- Referenced from [here](https://microk8s.io/docs/install-raspberry-pi)

#### NFS Prep

I am using an Synology DS413j with DSM 6.2.4-25556 Update 7 so the following steps will be inline with my Synology

#### Enable NFS
- Login to your Synology and go to `File Services`

![synology file services](images/how-to-bake-an-ortelius-pi/part02/01-syno-file-services-icon.png)

- On the `SMB/AFP/NFS` tab scroll until you see `NFS` and `enable NFS and enable NFSv4 support`

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

![synology file services](images/how-to-bake-an-ortelius-pi/part02/07-syno-file-sharing-next.png)




#### Conclusion

By this stage you should have three Pi 4 B's running with Ubuntu 22.04.4 LTS and MicroK8s. Stay tuned for part 3 where we will dive into optimising USB flash sticks for the best performance and stability and the installation of MicroK8s.

#### Disclaimer: Any brands I mention in this blog post series are not monetised. This is my home setup!
