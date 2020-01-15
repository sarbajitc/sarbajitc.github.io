---
layout: post
title:  "Kubernetes setup on lxd containers as host"
date:   2020-01-15 15:30:00
categories: kubernetes lxd dockerinlxc lxc
---

LXD is a container manager in Linux. It provides VM like experiences using lxc container runtime. Here is a step by step guide of setting up a kubernetes cluster on lxd containers.

### Setup
* VirtualBox
* A VM with Ubuntu 18.04
* VM has 4 CPU, 16GB RAM, 100GB disk (can be less)
* VM has 2 network interfaces. First a NAT adatper (for internet access), second is a host-only network adapter (for shell access from host)

### Pre-requisites to install kubernetes cluster on lxd containers
* Install latest package updates `apt-get update; apt-get upgrade`
* Turn off swap partition `sudo swapoff -a` and comment out the swap partition line from `/etc/fstab` file
* Initialize LXD daemon `sudo lxd init` and keep default answers for all prompts
* Run command `sudo lxd waitready`
* Add your user a/c to lxd group `sudo gpasswd -a "${USER}" lxd`
* Reboot the VM
