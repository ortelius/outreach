## How to bake an Ortelius Pi Part 1 | The Hardware

## Introduction

I recently started building an entire Cloud Native environment on three Raspberry Pi 4 B's with an old Synology DS413j (ARMv5 architecture) running the latest firmware update DSM 6.2.4-25556 Update 7 [Release Notes](https://www.synology.com/en-af/releaseNote/DSM) and so far its been quite a journey. In this blog post I would like to share my undertakings in a series of blog posts. First I will cover the Raspberry Pi hardware, NFS and setup and then move on to Canonicles MicroK8s ( Kubernetes), Traefik (Cloud Native Proxy and Load Balancer), Netdata for observability, ArgoCD, [DevPod](https://devpod.sh/) (Loft Labs) LocalStack (AWS Cloud at home) and Ortelius the ultimate evidence store.

Why Raspberry Pi's you ask, well first of all I live in Cape Town South Africa where we are experiencing some of the worst electricity outages in years thus we need to share electricity by taking turns through rotational blocks of time commonly know to locals as Load Shedding. We use an app like this one [Load Shedding App](https://play.google.com/store/apps/details?id=com.abisoft.loadsheddingnotifier&hl=en_ZA&gl=US) to inform ourselves when the next bout of load shedding will be hitting our area. Raspberry Pi 4 B's pack a punch with a Broadcom Quad Core ARMv8 processor and 8 GB ram. They are very light on electricy thus saving on cost and only require a single small UPS (uninterruptable power supply) to stay online. They are very mobile and take up extremely little space in my man cave.

#### My setup
- 3X Raspberry Pi 4 B | Please go to this link for the full hardware specs [Raspberry Pi 4 B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/)

![raspberry-pi-4b](images/raspberry-pi-4b.png)

- 3X 32GB minimum Micro SD Card (UHS-II: theoretical maximum transfer speeds up to 312MB/s) | Speed chart [here](https://www.kingston.com/en/blog/personal-storage/memory-card-speed-classes)

OR

- 3X 32GB USB 3 sticks but this comes with some caveats performance wise which I will discuss further on

- 1X Screen HDMI
- 1X Keyboard USB
- 1X [KVM Switch device](https://www.amazon.com/3-port-kvm-switch/s?k=3+port+kvm+switch)
- 1X 8 Port Switch
- 1X UPS (Uninterruptable Power Supply) for the Pi's and switch | Something like this [Mecer 650VA](https://mecer.co.za/mecer-line-interactive-ups/). Please note this is a South African brand of UPS but I am showing this for example purposes. The Mecer brand is extremely good and all my lead acid battery UPS's are from Mecer. I have a combination of the 650VA, 2000VA and 3000VA to keep me going (7 in all)

#### Storage
- 1X NAS device for the NFS storage which can be a virtual machine, laptop, old desktop or Synology type device. There are plenty options out there just do a internet search.
- 1X 650VA UPS

## Operating System
- Ubuntu Server 22.04 LTS x64

## Raspberry Pi Imaging Utility

The imaging utility will be used install Ubuntu onto your SD Card or USB flash drive.
- 1X [Raspberry Pi imaging Utility](https://www.raspberrypi.com/news/raspberry-pi-imager-imaging-utility/)


#### Disclaimer: Any brands I mention in this blog post series are not monetised. I do this for the love of tech and sharing with others.
