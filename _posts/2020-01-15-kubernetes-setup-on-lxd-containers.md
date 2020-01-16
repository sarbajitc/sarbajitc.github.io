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
* Once VM is running, create lxc containers to run kubernetes. I created `master`, `worker-1` and `worker-2` using e.g. `lxc launch ubuntu:18.04 master`

### Installing kubernetes on lxc containers (using kubeadm)
* Before kubernetes can be installed, there are following tweaks necessare in every lxc container created before -
  * From the VM shell run e.g. `lxc config edit master` and add following in config section -
  ```yaml
  config:
    linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,ip_vs,ip_vs_rr,ip_vs_wrr,ip_vs_sh,xt_conntrack,br_netfilter
    raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop= \nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw sys:rw"
    security.privileged: "true"
    security.nesting: "true"
  ```
* Login to container shell e.g. `lxc exec master /bin/bash` and run following commands -
  * Install linux image `apt-get install -y linux-image-$(uname -r)`
  * Create dummy entry for kmsg `echo 'L /dev/kmsg - - - - /dev/console' > /etc/tmpfiles.d/kmsg.conf`
  * Install docker runtime
  ```shell
  curl -fsSL https://get.docker.com -o get-docker.sh
  sh get-docker.sh
  ```
  * Install kubeadm, kubectl, kubelet packages
  ```shell
  sudo apt-get update && sudo apt-get install -y apt-transport-https curl
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
  cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
  deb https://apt.kubernetes.io/ kubernetes-xenial main
  EOF
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  ```
* Kubernetes [setup using kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) -
  * Login to master `lxc exec master /bin/bash` and run `kubeadm init` 
  * Once master node setup is completed configure kubectl tool -
  ```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
  * Check if kubectl command works `kubectl get nodes`
  * Login to worker nodes and join the cluster e.g. 
  ```shell
  kubeadm join 10.245.210.109:6443 --token tyuggx.dnhpmj4g8arvlrzn \
    --discovery-token-ca-cert-hash sha256:0c07e636f0ffc5eae6088600f1d8b0643f5cf423f38356feffc2bda69598378c
  ```
  * After the nodes join kuberenetes cluster check `kubectl get nodes` to confirm. Nodes will ne in `Not Ready` state at this point.
  * Install a CNI plugin of your choice. I used calico `kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml`
  * After CNI plugin is installed, cluster will become ready
  
  
  *P.S. Kubernetes cluster goes into bad state after rebooting the VM or the lxc containers*
